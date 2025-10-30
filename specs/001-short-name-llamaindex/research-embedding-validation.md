# Research: Embedding Model Validation for RAG Systems

**Research Task**: Investigate embedding model validation techniques to ensure consistency between document processing and query time
**Context**: Llamaindex + RamaLama RAG Demo (FR-006: Embedding Model Validation)
**Date**: 2025-10-30
**Researcher**: Stella (Staff Engineer)

---

## Executive Summary

**Decision**: Multi-layered validation approach combining database metadata storage, startup dimension checking, and fail-fast error handling with actionable messages.

**Rationale**: RAG systems are extremely sensitive to embedding model mismatches - using a different model at query time than at indexing time produces semantically meaningless similarity scores, leading to poor retrieval and confusing user experiences. A robust validation system prevents this failure mode entirely, making it the single most important reliability feature for a RAG demo.

**Key Insight**: The docs2db pipeline already stores comprehensive model metadata in the `models` table (name, dimensions, provider, description). We can leverage this existing infrastructure rather than building parallel metadata storage.

---

## Background: Why Embedding Consistency Matters

### The Problem
```
Document Processing Time:
  Document → Chunks → granite-30m-english (384 dim) → PGVector

Query Time (MISMATCH):
  Query → slate-125m-english (768 dim) → ???

Result: Dimension mismatch OR wrong semantic space = garbage results
```

### Real-World Impact
- **Silent Failure**: System appears to work but returns irrelevant results
- **Debugging Difficulty**: Users don't know whether poor results are due to model mismatch, bad chunking, or actual content gaps
- **Demo Credibility**: In a demo scenario, model mismatch destroys confidence in the technology

---

## Validation Approach Architecture

### Layer 1: Database Metadata Storage (EXISTING)

The docs2db pipeline already implements comprehensive model tracking:

```sql
-- Models table (from database.py lines 368-375)
CREATE TABLE IF NOT EXISTS models (
    id SERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,           -- e.g., "ibm-granite/granite-embedding-30m-english"
    dimensions INTEGER NOT NULL,          -- e.g., 384
    provider TEXT,                        -- e.g., "granite"
    description TEXT,                     -- e.g., "Embedding model: granite-30m-english"
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Embeddings reference models (lines 378-385)
CREATE TABLE IF NOT EXISTS embeddings (
    id SERIAL PRIMARY KEY,
    chunk_id INTEGER REFERENCES chunks(id) ON DELETE CASCADE,
    model_id INTEGER REFERENCES models(id) ON DELETE CASCADE,
    embedding VECTOR,  -- Dynamic dimension based on model
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(chunk_id, model_id)
);
```

**Key Features:**
- Foreign key relationship ensures every embedding is tied to a model
- `name` uses full model identifier (e.g., "ibm-granite/granite-embedding-30m-english")
- `dimensions` enables dimension validation without loading model
- `provider` helps map between short names (granite-30m-english) and full IDs

**Data Access Pattern** (from database.py lines 256-272):
```python
async def get_model_info(self, conn, model_id: int) -> Optional[dict]:
    """Get model information by ID."""
    result = await conn.execute(
        "SELECT id, name, dimensions, provider, description, created_at FROM models WHERE id = %s",
        [model_id],
    )
    row = await result.fetchone()
    if row:
        return {
            "id": row[0],
            "name": row[1],
            "dimensions": row[2],
            "provider": row[3],
            "description": row[4],
            "created_at": row[5],
        }
    return None
```

### Layer 2: Startup Validation Pattern

**When**: Before accepting first query (during Llamaindex service initialization)

**What to Validate**:
1. **Model Existence**: Database contains at least one embedding model
2. **Model Match**: Configured Llamaindex model matches database model(s)
3. **Dimension Match**: Embedding dimensions align between config and database
4. **Single Model Constraint**: Database uses only one model (demo simplification)

**Validation Query Pattern**:
```python
# Query to retrieve all models in database
async def get_database_models(conn) -> List[dict]:
    """Get all embedding models in the database."""
    result = await conn.execute("""
        SELECT
            m.name,
            m.dimensions,
            m.provider,
            COUNT(e.id) as embedding_count
        FROM models m
        LEFT JOIN embeddings e ON e.model_id = m.id
        GROUP BY m.id, m.name, m.dimensions, m.provider
        ORDER BY m.created_at
    """)

    models = []
    async for row in result:
        models.append({
            "name": row[0],
            "dimensions": row[1],
            "provider": row[2],
            "embedding_count": row[3]
        })
    return models
```

**Model Name Resolution**:
The docs2db system has short names ("granite-30m-english") and full model IDs ("ibm-granite/granite-embedding-30m-english"). The mapping is in EMBEDDING_CONFIGS (embeddings.py lines 454+):

```python
EMBEDDING_CONFIGS = {
    "granite-30m-english": {
        "model_id": "ibm-granite/granite-embedding-30m-english",
        "dimensions": 384,
        "provider": "granite",
    },
    "slate-125m-english-rtrvr-v2": {
        "model_id": "ibm/slate-125m-english-rtrvr-v2",
        "dimensions": 768,
        "provider": "watson",
    },
    # ... more models
}
```

**Resolution Strategy**:
```python
def resolve_model_name(configured_name: str) -> tuple[str, int, str]:
    """Resolve short model name to full ID, dimensions, provider.

    Returns:
        (full_model_id, dimensions, provider)

    Raises:
        ConfigurationError if model unknown
    """
    if configured_name not in EMBEDDING_CONFIGS:
        available = ", ".join(EMBEDDING_CONFIGS.keys())
        raise ConfigurationError(
            f"Unknown embedding model '{configured_name}'. "
            f"Available models: {available}"
        )

    config = EMBEDDING_CONFIGS[configured_name]
    return (
        config["model_id"],
        config["dimensions"],
        config["provider"]
    )
```

### Layer 3: Dimension Checking

**Validation Points**:

1. **Config vs Database**:
```python
async def validate_embedding_model(
    conn,
    configured_model_short_name: str
) -> tuple[bool, str]:
    """Validate that configured model matches database.

    Returns:
        (is_valid, error_message)
    """
    # Resolve configured model
    try:
        full_model_id, expected_dims, provider = resolve_model_name(
            configured_model_short_name
        )
    except ConfigurationError as e:
        return False, str(e)

    # Get database models
    db_models = await get_database_models(conn)

    # Check: Database has embeddings
    if not db_models:
        return False, (
            "Database contains no embedding models. "
            "Run docs2db pipeline to generate embeddings before starting demo."
        )

    # Check: Single model (demo constraint)
    if len(db_models) > 1:
        model_names = [m["name"] for m in db_models]
        return False, (
            f"Database contains multiple embedding models: {', '.join(model_names)}. "
            f"This demo supports single-model databases only. "
            f"Re-run docs2db pipeline with a single model."
        )

    # Check: Model name match
    db_model = db_models[0]
    if db_model["name"] != full_model_id:
        return False, (
            f"Embedding model mismatch detected:\n"
            f"  Database model    : {db_model['name']} ({db_model['dimensions']} dimensions)\n"
            f"  Configured model  : {full_model_id} ({expected_dims} dimensions)\n"
            f"\n"
            f"Fix: Update demo configuration to use '{configured_model_short_name}' "
            f"or re-run docs2db pipeline with the configured model.\n"
            f"\n"
            f"Database has {db_model['embedding_count']} embeddings. "
            f"Changing models requires re-embedding all documents."
        )

    # Check: Dimension match (redundant but catches corruption)
    if db_model["dimensions"] != expected_dims:
        return False, (
            f"Embedding dimension mismatch detected:\n"
            f"  Database  : {db_model['dimensions']} dimensions\n"
            f"  Configured: {expected_dims} dimensions\n"
            f"\n"
            f"This indicates database corruption or configuration error. "
            f"Contact support if this occurs."
        )

    # Check: Database not empty
    if db_model["embedding_count"] == 0:
        return False, (
            f"Database model '{db_model['name']}' exists but has no embeddings. "
            f"Run docs2db load to populate embeddings."
        )

    return True, ""
```

2. **Runtime Query Validation** (optional, performance trade-off):
```python
async def validate_query_embedding_dimensions(
    embedding_vector: List[float],
    expected_dimensions: int
) -> None:
    """Validate query embedding has expected dimensions.

    Raises:
        ValueError if dimensions don't match
    """
    actual_dims = len(embedding_vector)
    if actual_dims != expected_dimensions:
        raise ValueError(
            f"Query embedding dimension mismatch: "
            f"expected {expected_dimensions}, got {actual_dims}. "
            f"This indicates an internal error in the embedding model."
        )
```

### Layer 4: Error Handling & User Communication

**Fail-Fast Pattern**:
```python
class RagService:
    def __init__(self, config: dict):
        self.config = config
        self._validated = False
        self._db_connection = None
        self._model_info = None

    async def startup(self):
        """Startup sequence with validation."""
        # 1. Connect to database
        self._db_connection = await self._connect_database()

        # 2. Validate embedding model (GATE)
        is_valid, error_msg = await validate_embedding_model(
            self._db_connection,
            self.config["embedding_model"]
        )

        if not is_valid:
            logger.error(
                "Embedding model validation failed",
                error=error_msg
            )
            raise StartupValidationError(
                "Cannot start RAG service: Embedding model validation failed.\n\n"
                f"{error_msg}"
            )

        # 3. Cache model info for runtime use
        db_models = await get_database_models(self._db_connection)
        self._model_info = db_models[0]

        # 4. Initialize Llamaindex with validated model
        self._index = await self._initialize_llamaindex()

        self._validated = True
        logger.info(
            "RAG service started successfully",
            model=self._model_info["name"],
            dimensions=self._model_info["dimensions"],
            embeddings=self._model_info["embedding_count"]
        )

    async def query(self, query_text: str) -> dict:
        """Handle query (only after successful validation)."""
        if not self._validated:
            raise RuntimeError(
                "RAG service not validated. Call startup() first."
            )

        # Query processing...
```

**Health Endpoint Pattern**:
```python
@app.get("/health")
async def health_check(rag_service: RagService):
    """Health check with embedding model status."""
    if not rag_service._validated:
        return {
            "status": "unhealthy",
            "reason": "Embedding model validation not complete"
        }

    return {
        "status": "healthy",
        "components": {
            "database": "connected",
            "model": "validated",
            "embedding_model": {
                "name": rag_service._model_info["name"],
                "dimensions": rag_service._model_info["dimensions"],
                "embeddings": rag_service._model_info["embedding_count"]
            }
        }
    }
```

---

## Implementation Approach

### Step 1: Database Query Helper
Create utility to query model metadata from docs2db database.

**File**: `src/rag/validation.py`
```python
from typing import List, Optional, Dict, Tuple
import structlog
from psycopg import AsyncConnection

logger = structlog.get_logger()

async def get_database_models(conn: AsyncConnection) -> List[Dict]:
    """Query all embedding models from database."""
    # Implementation from Layer 2

async def validate_embedding_model(
    conn: AsyncConnection,
    configured_model_short_name: str
) -> Tuple[bool, str]:
    """Validate configured model matches database."""
    # Implementation from Layer 3
```

### Step 2: Startup Validation Integration
Integrate validation into Llamaindex service initialization.

**File**: `src/rag/service.py`
```python
from .validation import validate_embedding_model

class LlamaindexRagService:
    async def startup(self):
        """Initialize with model validation."""
        # 1. Database connection
        # 2. Validate model (GATE - fail if invalid)
        # 3. Initialize Llamaindex
        # 4. Mark as ready
```

### Step 3: Configuration Schema
Define configuration with explicit embedding model field.

**File**: `config.yaml`
```yaml
database:
  host: localhost
  port: 5432
  name: ragdb
  user: postgres
  password: postgres

embedding:
  model: granite-30m-english  # Short name from EMBEDDING_CONFIGS

rag:
  similarity_threshold: 0.7
  top_k: 5

llm:
  endpoint: http://localhost:8080
  model: phi-4-mini-instruct
```

### Step 4: Error Message Templates
Standardize user-facing error messages.

**File**: `src/rag/errors.py`
```python
class StartupValidationError(Exception):
    """Raised when startup validation fails."""
    pass

class EmbeddingModelMismatchError(StartupValidationError):
    """Raised when embedding model doesn't match database."""

    def __init__(self, db_model: str, config_model: str, db_dims: int, config_dims: int):
        super().__init__(
            f"Embedding model mismatch:\n"
            f"  Database: {db_model} ({db_dims}D)\n"
            f"  Config:   {config_model} ({config_dims}D)\n"
            f"\n"
            f"Action required: Update config.yaml or re-run docs2db pipeline"
        )
```

### Step 5: Health Endpoint
Expose validation status via health check.

**File**: `src/api/routes.py`
```python
@router.get("/health")
async def health(service: LlamaindexRagService = Depends(get_rag_service)):
    """Health check with model validation status."""
    # Implementation from Layer 4
```

---

## Database Schema Needs

**Good News**: No new tables or columns needed! The docs2db schema already provides everything required:

### Existing Schema (Sufficient)
```sql
-- Models table tracks all embedding models
models (
    id SERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,        -- Full model ID
    dimensions INTEGER NOT NULL,       -- For dimension validation
    provider TEXT,                     -- For model resolution
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
)

-- Embeddings table links chunks to models
embeddings (
    id SERIAL PRIMARY KEY,
    chunk_id INTEGER REFERENCES chunks(id),
    model_id INTEGER REFERENCES models(id),  -- FK to models table
    embedding VECTOR,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(chunk_id, model_id)
)
```

### Query Patterns
```sql
-- Get all models with embedding counts
SELECT
    m.name,
    m.dimensions,
    m.provider,
    m.description,
    COUNT(e.id) as embedding_count,
    MAX(e.created_at) as latest_embedding
FROM models m
LEFT JOIN embeddings e ON e.model_id = m.id
GROUP BY m.id, m.name, m.dimensions, m.provider, m.description
ORDER BY m.created_at;

-- Check for model by name
SELECT id, name, dimensions, provider
FROM models
WHERE name = 'ibm-granite/granite-embedding-30m-english';

-- Get embedding statistics for validation
SELECT
    COUNT(*) as total_embeddings,
    COUNT(DISTINCT chunk_id) as chunks_with_embeddings,
    AVG(array_length(embedding::real[], 1)) as avg_dimensions
FROM embeddings
WHERE model_id = $1;
```

---

## Alternatives Considered

### Alternative 1: Configuration File Metadata
Store model metadata in a JSON/YAML file alongside database dump.

**Pros**:
- Simple to implement
- Human-readable
- Version-controllable

**Cons**:
- **File/Database Desync**: File can drift from actual database content
- **Extra Maintenance**: Another file to manage and distribute
- **No Validation**: File could be edited incorrectly
- **Migration Complexity**: Moving database requires moving metadata file

**Verdict**: REJECTED - Adds complexity without reliability benefits

### Alternative 2: Model Fingerprinting
Hash the model weights and store fingerprint in database.

**Pros**:
- Cryptographically verifies exact model match
- Detects model version drift

**Cons**:
- **Overkill**: Model name + dimensions sufficient for this use case
- **Performance**: Computing hash requires loading full model
- **Storage**: Hashes are additional metadata
- **False Positives**: Different quantizations of same model have different hashes

**Verdict**: REJECTED - Unnecessary complexity for demo scope

### Alternative 3: Embedding Probe Query
Generate test embedding and compare with database embedding.

**Pros**:
- End-to-end validation
- Detects subtle incompatibilities

**Cons**:
- **Slow**: Requires model loading before validation
- **Unreliable**: Two embeddings of same text can differ slightly
- **Complex**: Need test document in database
- **False Negatives**: Small differences might not indicate real problems

**Verdict**: REJECTED - Too slow and unreliable for startup validation

### Alternative 4: Dimension-Only Checking
Only validate embedding dimensions match.

**Pros**:
- Simple to implement
- Fast validation
- Catches most errors

**Cons**:
- **Incomplete**: Different models can have same dimensions
  - Example: granite-30m-english (384D) vs NoInstruct-small-Embedding (384D)
  - Using wrong model gives semantically meaningless results
- **Silent Failure**: System runs but produces garbage results
- **Debugging Nightmare**: Users don't know why retrieval is poor

**Verdict**: REJECTED - Insufficient protection, defeats purpose of validation

### Alternative 5: Warn-and-Continue
Detect mismatch but allow system to start with warning.

**Pros**:
- Flexible for experimentation
- Doesn't block development

**Cons**:
- **Violates Fail-Fast Principle**: Error propagates to runtime
- **Poor UX**: Users get bad results and don't know why
- **Demo Failure**: In demo context, this destroys credibility
- **Support Burden**: Users will report "RAG doesn't work" issues

**Verdict**: REJECTED - Fail-fast is essential for demo reliability

---

## Error Messaging Best Practices

### Principle: Actionable Error Messages
Every error message must tell the user:
1. **What** is wrong
2. **Why** it matters
3. **How** to fix it

### Example Error Messages

**Scenario 1: Model Mismatch**
```
ERROR: Embedding model validation failed

Embedding model mismatch detected:
  Database model    : ibm/slate-125m-english-rtrvr-v2 (768 dimensions)
  Configured model  : ibm-granite/granite-embedding-30m-english (384 dimensions)

Impact: Query embeddings will be incompatible with indexed documents,
resulting in poor or random retrieval results.

Fix Options:
  1. Update config.yaml embedding.model to: slate-125m-english-rtrvr-v2
  2. Re-run docs2db pipeline with granite-30m-english model:
     $ docs2db embed --model granite-30m-english content/
     $ docs2db load --model granite-30m-english content/

Database contains 1,247 embeddings. Changing models requires re-embedding all documents.
```

**Scenario 2: No Embeddings**
```
ERROR: Embedding model validation failed

Database contains no embeddings.

Impact: RAG system cannot retrieve documents without embeddings.

Fix: Run docs2db pipeline to generate embeddings:
  $ docs2db ingest content/
  $ docs2db chunk content/
  $ docs2db embed --model granite-30m-english content/
  $ docs2db load --model granite-30m-english content/

See README.md for detailed pipeline instructions.
```

**Scenario 3: Multiple Models**
```
ERROR: Embedding model validation failed

Database contains multiple embedding models:
  - ibm-granite/granite-embedding-30m-english (1,247 embeddings)
  - ibm/slate-125m-english-rtrvr-v2 (850 embeddings)

This demo supports single-model databases only.

Fix: Re-initialize database with a single model:
  1. Backup existing database: $ make db-dump
  2. Drop database: $ make db-down && make db-up
  3. Re-run docs2db pipeline with one model:
     $ docs2db load --model granite-30m-english content/
```

**Scenario 4: Unknown Model**
```
ERROR: Configuration validation failed

Unknown embedding model: 'my-custom-model'

Available models:
  - granite-30m-english (384D, ibm-granite/granite-embedding-30m-english)
  - slate-125m-english-rtrvr-v2 (768D, ibm/slate-125m-english-rtrvr-v2)
  - e5-small-v2 (384D, intfloat/e5-small-v2)
  - NoInstruct-small-Embedding-v0 (384D, avsolatorio/NoInstruct-small-Embedding-v0)

Fix: Update config.yaml embedding.model to one of the available models.

To add custom models, see docs2db embeddings.py documentation.
```

### Error Message Template
```python
ERROR_TEMPLATE = """
ERROR: {error_category}

{problem_description}

Impact: {why_this_matters}

Fix: {how_to_resolve}

{additional_context}
"""
```

---

## Testing Strategy

### Unit Tests
```python
async def test_validate_model_success():
    """Test validation passes with matching model."""
    # Setup: Database with granite-30m-english embeddings
    # Config: granite-30m-english
    # Assert: validation passes

async def test_validate_model_mismatch_name():
    """Test validation fails with wrong model name."""
    # Setup: Database with granite-30m-english
    # Config: slate-125m-english-rtrvr-v2
    # Assert: validation fails with clear error

async def test_validate_model_mismatch_dimensions():
    """Test validation fails with dimension mismatch."""
    # Setup: Corrupt database (wrong dimensions)
    # Config: granite-30m-english
    # Assert: validation fails with corruption error

async def test_validate_model_no_embeddings():
    """Test validation fails with empty database."""
    # Setup: Database with model but no embeddings
    # Assert: validation fails with "no embeddings" error

async def test_validate_model_multiple_models():
    """Test validation fails with multiple models."""
    # Setup: Database with 2+ models
    # Assert: validation fails with "multiple models" error
```

### Integration Tests
```python
async def test_startup_with_valid_model():
    """Test service starts successfully with valid model."""
    # Setup: Valid database + matching config
    # Assert: Service starts, health endpoint returns healthy

async def test_startup_with_invalid_model_fails():
    """Test service fails to start with invalid model."""
    # Setup: Database with granite, config with slate
    # Assert: Service raises StartupValidationError

async def test_query_requires_validation():
    """Test queries fail before validation."""
    # Setup: Service not validated
    # Assert: Query raises RuntimeError
```

### Manual Testing Checklist
- [ ] Start service with matching model → Success
- [ ] Start service with mismatched model → Clear error message
- [ ] Start service with empty database → Clear error message
- [ ] Start service with multi-model database → Clear error message
- [ ] Check `/health` endpoint shows model info
- [ ] Check `/stats` endpoint shows embedding counts
- [ ] Verify error messages are actionable

---

## Performance Considerations

### Validation Overhead
- **Startup**: 1-2 SQL queries (~10-50ms)
- **Runtime**: Zero overhead (validation cached after startup)
- **Trade-off**: Worth it - prevents catastrophic failure mode

### Optimization Opportunities
1. **Cache Model Info**: Store validated model info in memory
2. **Skip Runtime Checks**: Only validate at startup, not per query
3. **Connection Pooling**: Reuse database connection for validation queries

### Scalability
For production (beyond demo scope):
- **Multi-Model Support**: Remove single-model constraint
- **Model Registry**: Track which chunks use which models
- **Per-Query Model Selection**: Allow queries to specify target model
- **Model Versioning**: Track model version changes over time

---

## Security Considerations

### SQL Injection Protection
Use parameterized queries for all database operations:
```python
# GOOD
await conn.execute(
    "SELECT * FROM models WHERE name = %s",
    [model_name]
)

# BAD
await conn.execute(
    f"SELECT * FROM models WHERE name = '{model_name}'"
)
```

### Error Message Information Disclosure
Error messages reveal database schema and model names - acceptable for demo, but production systems should:
- Log detailed errors server-side
- Return generic errors to clients
- Require authentication for health endpoint details

---

## Documentation Requirements

### README Section: Embedding Model Configuration
```markdown
## Embedding Model Configuration

The RAG demo requires exact matching between:
1. The embedding model used in docs2db pipeline
2. The embedding model configured in config.yaml

### Checking Your Database Model

```bash
# Check which model is in your database
make db-stats
```

Look for the "Embedding model details" section.

### Configuring the Demo

Edit `config.yaml`:
```yaml
embedding:
  model: granite-30m-english  # Must match database model
```

### Troubleshooting Model Mismatches

If you see "Embedding model mismatch" error:

**Option 1: Update Config (Fast)**
Change config.yaml to match database model.

**Option 2: Re-run Pipeline (Slow but Clean)**
Re-embed all documents with desired model:
```bash
cd /path/to/docs2db
docs2db embed --model granite-30m-english content/
docs2db load --model granite-30m-english content/
```

### Supported Models

See docs2db `embeddings.py` for full list. Common models:
- `granite-30m-english`: 384D, fast, good quality
- `slate-125m-english-rtrvr-v2`: 768D, higher quality, slower
- `e5-small-v2`: 384D, general purpose

**Important**: Different models have different dimensions and semantic spaces.
You cannot mix models - the entire database must use one model.
```

### Architecture Documentation
Include validation in system diagrams:
```
┌─────────────────────────────────────────────────────────────┐
│ RAG Service Startup                                         │
├─────────────────────────────────────────────────────────────┤
│ 1. Load Configuration                                        │
│ 2. Connect to Database                                       │
│ 3. Validate Embedding Model ◄─── GATE (fail-fast)          │
│    ├─ Query models table                                    │
│    ├─ Check model name match                                │
│    ├─ Check dimension match                                 │
│    └─ Check embedding count > 0                             │
│ 4. Initialize Llamaindex (only if validation passed)        │
│ 5. Mark service as ready                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## References

### Docs2db Codebase
- **database.py lines 368-385**: Models and embeddings table schema
- **database.py lines 217-248**: `insert_model()` and `get_model_info()` methods
- **embeddings.py lines 454+**: EMBEDDING_CONFIGS mapping
- **embeddings.py lines 45-67**: Embedding staleness checking pattern

### Best Practices
- **Fail-Fast Pattern**: Detect errors at startup, not runtime
- **Actionable Errors**: Every error message includes fix instructions
- **Defense in Depth**: Multiple validation layers (config, database, dimensions)
- **Explicit Over Implicit**: Store model metadata explicitly, don't infer

### Related Patterns
- **Database Migration Safety**: Similar validation for schema version compatibility
- **API Versioning**: Similar validation for API compatibility
- **Feature Flags**: Similar startup validation for required features

---

## Conclusion

**Recommended Implementation**:
1. Use existing `models` table for metadata storage (no schema changes needed)
2. Implement startup validation with fail-fast error handling
3. Provide actionable error messages with fix instructions
4. Cache validated model info for runtime efficiency
5. Expose validation status via `/health` endpoint

**Why This Works**:
- **Leverages Existing Infrastructure**: docs2db already tracks models
- **Fail-Fast Reliability**: Catches errors before accepting queries
- **Clear User Experience**: Actionable errors guide users to fixes
- **Zero Runtime Overhead**: Validation only at startup
- **Future-Proof**: Easy to extend for multi-model support

**Critical Success Factor**: Error messages must be excellent. In a demo context, a confusing error message is worse than no validation - users will blame the technology rather than the configuration. Every error message must clearly explain what's wrong and exactly how to fix it.

This validation approach transforms embedding model mismatch from a silent, catastrophic failure into a loud, helpful startup failure that guides users to correct configuration. It's the difference between "RAG doesn't work" and "Here's exactly what's misconfigured and how to fix it."

---

**Next Steps**:
1. Implement `validation.py` module with validation logic
2. Integrate into Llamaindex service startup sequence
3. Add validation tests to test suite
4. Document validation in README troubleshooting section
5. Create example error message documentation
