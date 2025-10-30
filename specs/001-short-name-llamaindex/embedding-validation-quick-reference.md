# Embedding Model Validation - Quick Reference Card

**For**: Implementation team working on Llamaindex RAG demo
**Purpose**: Fast lookup for validation implementation details

---

## TL;DR

✅ **DO**: Validate embedding model at startup, fail-fast with actionable errors
❌ **DON'T**: Allow service to start with mismatched models
💡 **KEY**: Use existing `models` table, no schema changes needed

---

## Critical SQL Queries

### Get Database Models
```sql
SELECT
    m.name,
    m.dimensions,
    m.provider,
    COUNT(e.id) as embedding_count
FROM models m
LEFT JOIN embeddings e ON e.model_id = m.id
GROUP BY m.id, m.name, m.dimensions, m.provider
ORDER BY m.created_at;
```

### Check Specific Model
```sql
SELECT id, name, dimensions
FROM models
WHERE name = %s;  -- e.g., 'ibm-granite/granite-embedding-30m-english'
```

---

## Validation Checklist

Run these checks in order at startup:

1. ✅ **Config Valid**: Model exists in `EMBEDDING_CONFIGS`
2. ✅ **Database Has Models**: `SELECT COUNT(*) FROM models > 0`
3. ✅ **Single Model**: Database has exactly 1 model (demo constraint)
4. ✅ **Name Match**: Database model name == resolved config model name
5. ✅ **Dimension Match**: Database dimensions == config dimensions
6. ✅ **Has Embeddings**: `embedding_count > 0`

**If ANY check fails**: Raise `StartupValidationError` with actionable message

---

## Model Name Resolution

```python
# Config has short name
config["embedding"]["model"] = "granite-30m-english"

# Resolve to full model ID
from docs2db.embeddings import EMBEDDING_CONFIGS
config_entry = EMBEDDING_CONFIGS["granite-30m-english"]
full_model_id = config_entry["model_id"]
# Result: "ibm-granite/granite-embedding-30m-english"

# Database stores full model ID
# Query: SELECT name FROM models WHERE name = 'ibm-granite/granite-embedding-30m-english'
```

**Common Short Names**:
- `granite-30m-english` → `ibm-granite/granite-embedding-30m-english` (384D)
- `slate-125m-english-rtrvr-v2` → `ibm/slate-125m-english-rtrvr-v2` (768D)
- `e5-small-v2` → `intfloat/e5-small-v2` (384D)

---

## Code Template

```python
# src/rag/validation.py
from typing import List, Dict, Tuple
from psycopg import AsyncConnection
from docs2db.embeddings import EMBEDDING_CONFIGS

async def validate_embedding_model(
    conn: AsyncConnection,
    configured_model_short_name: str
) -> Tuple[bool, str]:
    """Validate configured model matches database.

    Returns: (is_valid, error_message)
    """
    # 1. Resolve short name to full ID
    if configured_model_short_name not in EMBEDDING_CONFIGS:
        available = ", ".join(EMBEDDING_CONFIGS.keys())
        return False, f"Unknown model '{configured_model_short_name}'. Available: {available}"

    config = EMBEDDING_CONFIGS[configured_model_short_name]
    full_model_id = config["model_id"]
    expected_dims = config["dimensions"]

    # 2. Query database models
    result = await conn.execute("""
        SELECT m.name, m.dimensions, COUNT(e.id) as embedding_count
        FROM models m
        LEFT JOIN embeddings e ON e.model_id = m.id
        GROUP BY m.id, m.name, m.dimensions
    """)

    models = []
    async for row in result:
        models.append({
            "name": row[0],
            "dimensions": row[1],
            "embedding_count": row[2]
        })

    # 3. Validation checks
    if len(models) == 0:
        return False, "Database contains no embedding models. Run docs2db pipeline."

    if len(models) > 1:
        names = [m["name"] for m in models]
        return False, f"Database has multiple models: {names}. Demo supports single model only."

    db_model = models[0]

    if db_model["name"] != full_model_id:
        return False, (
            f"Model mismatch:\n"
            f"  Database: {db_model['name']} ({db_model['dimensions']}D)\n"
            f"  Config: {full_model_id} ({expected_dims}D)\n"
            f"Fix: Update config.yaml or re-run docs2db pipeline"
        )

    if db_model["dimensions"] != expected_dims:
        return False, f"Dimension mismatch: DB={db_model['dimensions']}, Config={expected_dims}"

    if db_model["embedding_count"] == 0:
        return False, f"Model '{db_model['name']}' has no embeddings. Run docs2db load."

    return True, ""

# src/rag/service.py
class RagService:
    async def startup(self):
        conn = await self._connect_db()

        is_valid, error_msg = await validate_embedding_model(
            conn,
            self.config["embedding"]["model"]
        )

        if not is_valid:
            raise StartupValidationError(
                f"Embedding validation failed:\n\n{error_msg}"
            )

        # Continue with Llamaindex initialization...
```

---

## Error Message Template

```python
# For model mismatch errors
ERROR_MISMATCH = """
ERROR: Embedding model validation failed

Embedding model mismatch detected:
  Database model    : {db_model_name} ({db_dims} dimensions)
  Configured model  : {config_model_name} ({config_dims} dimensions)

Impact: Query embeddings will be incompatible with indexed documents,
resulting in poor or random retrieval results.

Fix Options:
  1. Update config.yaml embedding.model to: {short_name_from_db}
  2. Re-run docs2db pipeline with configured model:
     $ docs2db embed --model {configured_short_name} content/
     $ docs2db load --model {configured_short_name} content/

Database has {embedding_count} embeddings. Changing models requires re-embedding all documents.
"""

# For empty database errors
ERROR_NO_EMBEDDINGS = """
ERROR: Embedding model validation failed

Database contains no embeddings.

Impact: RAG system cannot retrieve documents without embeddings.

Fix: Run docs2db pipeline to generate embeddings:
  $ docs2db ingest content/
  $ docs2db chunk content/
  $ docs2db embed --model {configured_model} content/
  $ docs2db load --model {configured_model} content/

See README.md for detailed pipeline instructions.
"""
```

---

## Health Endpoint

```python
@app.get("/health")
async def health(service: RagService = Depends(get_service)):
    if not service._validated:
        return JSONResponse(
            status_code=503,
            content={
                "status": "unhealthy",
                "reason": "Embedding model validation not complete"
            }
        )

    return {
        "status": "healthy",
        "components": {
            "database": "connected",
            "model": "validated"
        },
        "embedding_model": {
            "name": service._model_info["name"],
            "dimensions": service._model_info["dimensions"],
            "embeddings": service._model_info["embedding_count"]
        }
    }
```

---

## Testing Scenarios

```python
# Unit test cases
async def test_validate_success():
    # Setup: DB with granite-30m, config with granite-30m
    # Assert: validation passes

async def test_validate_mismatch():
    # Setup: DB with granite-30m, config with slate-125m
    # Assert: validation fails with clear error

async def test_validate_no_embeddings():
    # Setup: DB with model but no embeddings
    # Assert: validation fails with "no embeddings" error

async def test_validate_multiple_models():
    # Setup: DB with 2+ models
    # Assert: validation fails with "multiple models" error

# Integration test cases
async def test_startup_valid():
    # Setup: Valid DB + config
    # Assert: Service starts, /health returns healthy

async def test_startup_invalid():
    # Setup: Mismatched DB + config
    # Assert: Service raises StartupValidationError, /health returns 503
```

---

## Configuration Example

```yaml
# config.yaml
database:
  host: localhost
  port: 5432
  name: ragdb
  user: postgres
  password: postgres

embedding:
  model: granite-30m-english  # CRITICAL: Must match docs2db pipeline model

rag:
  similarity_threshold: 0.7
  top_k: 5
```

---

## Common Pitfalls

❌ **Don't**: Use different models for indexing vs querying
❌ **Don't**: Skip validation for "faster" startup
❌ **Don't**: Return generic errors - be specific
❌ **Don't**: Allow queries before validation completes

✅ **Do**: Validate at startup before accepting queries
✅ **Do**: Cache model info after validation (zero runtime overhead)
✅ **Do**: Provide actionable error messages with fix commands
✅ **Do**: Expose validation status via /health endpoint

---

## Performance Notes

- **Startup**: 1-2 SQL queries (~10-50ms total)
- **Runtime**: Zero overhead (validation cached)
- **Memory**: ~1KB for cached model metadata

**Trade-off**: 50ms startup overhead prevents hours of debugging time

---

## File Locations

```
demos/llamaindex-ramalama/
├── src/
│   ├── rag/
│   │   ├── validation.py      # ← Validation logic here
│   │   ├── service.py          # ← Call validation in startup()
│   │   └── errors.py           # ← StartupValidationError
│   ├── api/
│   │   └── routes.py           # ← /health endpoint
│   └── config/
│       └── config.yaml         # ← embedding.model config
└── tests/
    ├── unit/
    │   └── test_validation.py  # ← Validation unit tests
    └── integration/
        └── test_startup.py     # ← Startup integration tests
```

---

## Quick Debugging

### Check what's in the database
```bash
make db-stats
# Look for "Embedding model details" section
```

### Check database directly
```sql
\c ragdb
SELECT m.name, m.dimensions, COUNT(e.id) FROM models m LEFT JOIN embeddings e ON e.model_id = m.id GROUP BY m.id;
```

### Check config
```bash
grep "model:" config.yaml
```

### Test validation manually
```python
from docs2db.embeddings import EMBEDDING_CONFIGS
print(EMBEDDING_CONFIGS["granite-30m-english"]["model_id"])
# Should print: ibm-granite/granite-embedding-30m-english
```

---

## When Validation Fails

1. **Check Database Model**: `make db-stats` - see what model is in DB
2. **Check Config**: `cat config.yaml` - see configured model
3. **Read Error Message**: It tells you exactly what's wrong and how to fix it
4. **Choose Fix**:
   - **Option A (Fast)**: Update config.yaml to match database
   - **Option B (Clean)**: Re-run docs2db pipeline with desired model

---

## Questions?

See full documentation:
- [research-embedding-validation.md](./research-embedding-validation.md) - Complete research
- [embedding-validation-summary.md](./embedding-validation-summary.md) - Executive summary

---

**Remember**: Model validation is the most important reliability feature for a RAG demo. Get the error messages right!
