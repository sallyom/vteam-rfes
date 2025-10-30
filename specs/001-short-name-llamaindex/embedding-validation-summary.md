# Embedding Model Validation for RAG Systems - Executive Summary

**Research Context**: Llamaindex + RamaLama RAG Demo
**Date**: 2025-10-30
**Researcher**: Stella (Staff Engineer)

---

## Decision

**Multi-layered validation approach** combining database metadata storage, startup dimension checking, and fail-fast error handling with actionable error messages.

The system will:
1. Query the existing `models` table in the docs2db database at startup
2. Validate configured embedding model matches database model (name + dimensions)
3. Fail immediately with clear, actionable error if mismatch detected
4. Cache validated model info for runtime use with zero overhead
5. Expose validation status via `/health` endpoint

---

## Rationale

**Why this is most reliable:**

1. **Leverages Existing Infrastructure**: The docs2db pipeline already stores comprehensive model metadata in the `models` table (name, dimensions, provider, description). No new database schema needed.

2. **Fail-Fast Principle**: Validation occurs at startup before accepting any queries. Model mismatches are catastrophic errors that produce meaningless similarity scores - catching them early prevents confusing user experiences.

3. **Zero Runtime Overhead**: Validation happens once at startup and results are cached. No performance impact on query processing.

4. **Actionable Error Messages**: Every error message explains what's wrong, why it matters, and exactly how to fix it - critical for a demo that others will learn from.

5. **Defense in Depth**: Multiple validation layers:
   - Configuration validation (model exists in EMBEDDING_CONFIGS)
   - Database presence validation (embeddings exist)
   - Name matching validation (model IDs match)
   - Dimension validation (catches corruption)
   - Single-model constraint (simplifies demo)

**Critical Insight**: RAG systems are extremely sensitive to embedding model mismatches. Using a different model at query time than at indexing time produces semantically meaningless similarity scores, leading to poor retrieval that users can't debug. This validation prevents the single most common and confusing RAG failure mode.

---

## Alternatives Considered

### 1. Configuration File Metadata
Store model metadata in JSON/YAML file alongside database.

**Rejected**: File can desync from database, adds maintenance burden, no validation guarantees.

### 2. Model Fingerprinting
Hash model weights and store cryptographic fingerprint.

**Rejected**: Overkill for this use case, slow (requires loading model), storage overhead, false positives from quantization differences.

### 3. Embedding Probe Query
Generate test embedding and compare with database embedding.

**Rejected**: Too slow for startup validation, unreliable (embeddings have small variations), needs test document in database.

### 4. Dimension-Only Checking
Only validate embedding dimensions match, ignore model names.

**Rejected**: Insufficient - different models can have same dimensions (e.g., granite-30m-english and NoInstruct-small-Embedding both have 384D). Would allow silent failures with meaningless results.

### 5. Warn-and-Continue
Detect mismatch but allow system to start with warning.

**Rejected**: Violates fail-fast principle, produces confusing runtime errors, destroys demo credibility. Users would report "RAG doesn't work" without understanding why.

---

## Implementation Approach

### Step 1: Database Query Helper (`src/rag/validation.py`)

```python
async def get_database_models(conn: AsyncConnection) -> List[Dict]:
    """Query all embedding models from database with embedding counts."""
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
    # Return list of model dicts

async def validate_embedding_model(
    conn: AsyncConnection,
    configured_model_short_name: str
) -> Tuple[bool, str]:
    """Validate configured model matches database.

    Returns: (is_valid, error_message)
    """
    # 1. Resolve configured model from EMBEDDING_CONFIGS
    # 2. Query database models
    # 3. Check: Database has embeddings
    # 4. Check: Single model (demo constraint)
    # 5. Check: Model name match
    # 6. Check: Dimension match
    # 7. Check: Embedding count > 0
    # Return (True, "") or (False, actionable_error_message)
```

### Step 2: Startup Validation Integration (`src/rag/service.py`)

```python
class LlamaindexRagService:
    def __init__(self, config: dict):
        self._validated = False
        self._model_info = None

    async def startup(self):
        """Initialize with model validation gate."""
        # 1. Connect to database
        conn = await self._connect_database()

        # 2. Validate embedding model (GATE - fail if invalid)
        is_valid, error_msg = await validate_embedding_model(
            conn,
            self.config["embedding_model"]
        )

        if not is_valid:
            raise StartupValidationError(
                "Cannot start RAG service: Embedding model validation failed.\n\n"
                f"{error_msg}"
            )

        # 3. Cache model info for runtime use
        models = await get_database_models(conn)
        self._model_info = models[0]

        # 4. Initialize Llamaindex with validated model
        self._index = await self._initialize_llamaindex()

        self._validated = True

    async def query(self, query_text: str):
        """Handle query (only after successful validation)."""
        if not self._validated:
            raise RuntimeError("Service not validated. Call startup() first.")
        # Process query...
```

### Step 3: Configuration Schema (`config.yaml`)

```yaml
database:
  host: localhost
  port: 5432
  name: ragdb
  user: postgres
  password: postgres

embedding:
  model: granite-30m-english  # Short name from EMBEDDING_CONFIGS
                               # Must match model used in docs2db pipeline

rag:
  similarity_threshold: 0.7
  top_k: 5

llm:
  endpoint: http://localhost:8080
  model: phi-4-mini-instruct
```

### Step 4: Health Endpoint (`src/api/routes.py`)

```python
@router.get("/health")
async def health(service: LlamaindexRagService = Depends(get_rag_service)):
    """Health check with model validation status."""
    if not service._validated:
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
                "name": service._model_info["name"],
                "dimensions": service._model_info["dimensions"],
                "embeddings": service._model_info["embedding_count"]
            }
        }
    }
```

### Step 5: Error Classes (`src/rag/errors.py`)

```python
class StartupValidationError(Exception):
    """Raised when startup validation fails."""
    pass

class EmbeddingModelMismatchError(StartupValidationError):
    """Raised when embedding model doesn't match database."""

    def __init__(self, db_model: str, config_model: str,
                 db_dims: int, config_dims: int, embedding_count: int):
        super().__init__(
            f"Embedding model mismatch detected:\n"
            f"  Database model    : {db_model} ({db_dims} dimensions)\n"
            f"  Configured model  : {config_model} ({config_dims} dimensions)\n"
            f"\n"
            f"Impact: Query embeddings will be incompatible with indexed documents,\n"
            f"resulting in poor or random retrieval results.\n"
            f"\n"
            f"Fix Options:\n"
            f"  1. Update config.yaml embedding.model to match database\n"
            f"  2. Re-run docs2db pipeline with configured model:\n"
            f"     $ docs2db embed --model <model> content/\n"
            f"     $ docs2db load --model <model> content/\n"
            f"\n"
            f"Database has {embedding_count} embeddings. "
            f"Changing models requires re-embedding all documents."
        )
```

---

## Database Schema Needs

**Good news**: No new tables or columns needed!

The docs2db pipeline already provides everything required:

### Existing Schema (Sufficient)

```sql
-- Models table (tracks all embedding models)
CREATE TABLE models (
    id SERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,           -- e.g., "ibm-granite/granite-embedding-30m-english"
    dimensions INTEGER NOT NULL,          -- e.g., 384
    provider TEXT,                        -- e.g., "granite"
    description TEXT,                     -- e.g., "Embedding model: granite-30m-english"
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Embeddings table (links chunks to models)
CREATE TABLE embeddings (
    id SERIAL PRIMARY KEY,
    chunk_id INTEGER REFERENCES chunks(id) ON DELETE CASCADE,
    model_id INTEGER REFERENCES models(id) ON DELETE CASCADE,
    embedding VECTOR,  -- Dynamic dimension based on model
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(chunk_id, model_id)
);
```

### Key Query Patterns

```sql
-- Get all models with embedding counts (validation query)
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

-- Check for specific model by name
SELECT id, name, dimensions, provider
FROM models
WHERE name = 'ibm-granite/granite-embedding-30m-english';

-- Verify embedding dimensions (corruption check)
SELECT
    COUNT(*) as total_embeddings,
    AVG(array_length(embedding::real[], 1)) as avg_dimensions
FROM embeddings
WHERE model_id = $1;
```

---

## Error Messaging

### Principle: Every error must be actionable

Each error message contains:
1. **What** is wrong
2. **Why** it matters (impact on system)
3. **How** to fix it (specific commands)

### Example Error Messages

**Scenario 1: Model Name Mismatch**
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
     $ cd /path/to/docs2db
     $ docs2db embed --model granite-30m-english content/
     $ docs2db load --model granite-30m-english content/

Database contains 1,247 embeddings. Changing models requires re-embedding all documents.
```

**Scenario 2: No Embeddings in Database**
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

**Scenario 3: Multiple Models Detected**
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

**Scenario 4: Unknown Model in Config**
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

Fix: {step_by_step_resolution}

{additional_context}
"""
```

---

## Testing Strategy

### Unit Tests
- `test_validate_model_success()` - Validation passes with matching model
- `test_validate_model_mismatch_name()` - Fails with wrong model name
- `test_validate_model_mismatch_dimensions()` - Fails with dimension mismatch
- `test_validate_model_no_embeddings()` - Fails with empty database
- `test_validate_model_multiple_models()` - Fails with multiple models

### Integration Tests
- `test_startup_with_valid_model()` - Service starts successfully
- `test_startup_with_invalid_model_fails()` - Service raises StartupValidationError
- `test_query_requires_validation()` - Queries fail before validation
- `test_health_endpoint_shows_model_info()` - Health endpoint exposes validation status

### Manual Testing Checklist
- [ ] Start with matching model → Success
- [ ] Start with mismatched model → Clear error message
- [ ] Start with empty database → Clear error message
- [ ] Start with multi-model database → Clear error message
- [ ] Verify `/health` shows model info when healthy
- [ ] Verify `/health` shows unhealthy before validation
- [ ] Verify error messages are actionable

---

## Performance Impact

**Startup**: 1-2 SQL queries (~10-50ms) - negligible overhead
**Runtime**: Zero overhead - validation cached after startup
**Memory**: ~1KB for cached model metadata

**Trade-off Analysis**: The 50ms startup overhead is absolutely worth it to prevent catastrophic failures that waste hours of debugging time.

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

**Option 2: Re-run Pipeline (Clean)**
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

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ RAG Service Startup Sequence                                │
├─────────────────────────────────────────────────────────────┤
│ 1. Load Configuration (config.yaml)                         │
│ 2. Connect to Database (PGVector)                           │
│ 3. Validate Embedding Model ◄─── GATE (fail-fast)          │
│    ├─ Query models table                                    │
│    ├─ Check model name matches config                       │
│    ├─ Check dimensions match config                         │
│    ├─ Check embedding count > 0                             │
│    └─ Enforce single-model constraint                       │
│ 4. Initialize Llamaindex (only if validation passed)        │
│ 5. Mark service as ready                                    │
│ 6. Accept queries via /query endpoint                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Use existing docs2db infrastructure** - No new database schema needed
2. **Fail-fast at startup** - Catch errors before accepting queries
3. **Actionable error messages** - Every error includes fix instructions
4. **Zero runtime overhead** - Validation happens once, results cached
5. **Demo-appropriate scope** - Single-model constraint simplifies validation

**Critical Success Factor**: Error message quality is paramount. In a demo context, a confusing error message is worse than no validation - users will blame the technology rather than configuration. Every error message must clearly explain what's wrong and exactly how to fix it.

**Bottom Line**: This validation approach transforms embedding model mismatch from a silent, catastrophic failure into a loud, helpful startup failure that guides users to correct configuration. It's the difference between "RAG doesn't work" and "Here's exactly what's misconfigured and how to fix it."

---

## References

- **Full Research Document**: [research-embedding-validation.md](./research-embedding-validation.md)
- **Docs2db Database Schema**: `docs2db/src/docs2db/database.py` lines 368-385
- **Embedding Configs**: `docs2db/src/docs2db/embeddings.py` lines 454+
- **RFE Requirement**: FR-006 (Embedding Model Validation)
