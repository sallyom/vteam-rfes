# Data Model: LlamaIndex + pgvectordb RAG Demo

**Feature**: LlamaIndex + pgvectordb RAG Demo
**Branch**: `001-llamaindex-rag-demo`
**Date**: 2025-10-21

This document describes the data entities and their relationships in the RAG demonstration system.

---

## Entity Overview

The RAG demo operates on two data layers:
1. **docs2db Layer**: Normalized relational schema (source of truth)
2. **LlamaIndex Layer**: Denormalized view for query optimization

---

## 1. docs2db Schema Entities (Source of Truth)

### 1.1 Document Entity

Represents original files ingested into the system.

**Table**: `documents`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | SERIAL | PRIMARY KEY | Unique document identifier |
| path | TEXT | UNIQUE NOT NULL | Absolute file path of source document |
| filename | TEXT | NOT NULL | File name with extension |
| content_type | TEXT | | MIME type (e.g., "application/pdf", "text/markdown") |
| file_size | BIGINT | | File size in bytes |
| last_modified | TIMESTAMP WITH TIME ZONE | | Last modification timestamp from filesystem |
| chunks_file_path | TEXT | | Optional path to exported chunks file |
| created_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Ingestion timestamp |
| updated_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Last update timestamp |

**Relationships**:
- One document has many chunks (1:N)

**Validation Rules**:
- `path` must be unique (prevents duplicate ingestion)
- `filename` must not be empty
- `file_size` must be non-negative
- `last_modified` must not be in the future

**State Transitions**:
- Created → document ingested and record created
- Updated → document re-ingested, chunks regenerated
- Deleted → cascades to chunks, embeddings (via foreign keys)

---

### 1.2 Chunk Entity

Represents segmented portions of documents created during ingestion.

**Table**: `chunks`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | SERIAL | PRIMARY KEY | Unique chunk identifier |
| document_id | INTEGER | REFERENCES documents(id) | Parent document |
| chunk_index | INTEGER | NOT NULL | Sequential position in document (0-based) |
| text | TEXT | NOT NULL | Chunk text content |
| metadata | JSONB | | Additional chunk metadata (e.g., page number, section) |
| created_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Creation timestamp |

**Indexes**:
- UNIQUE(document_id, chunk_index) - ensures no duplicate chunks
- INDEX on document_id - fast document → chunks lookup

**Relationships**:
- Many chunks belong to one document (N:1)
- One chunk has many embeddings (1:N) - one per embedding model

**Validation Rules**:
- `chunk_index` must be non-negative
- `text` must not be empty
- `chunk_index` must be unique within a document
- chunks are ordered by `chunk_index` within a document

**Metadata Structure** (JSONB):
```json
{
  "page_number": 5,
  "section": "Introduction",
  "token_count": 512,
  "char_count": 2048,
  "source_start": 1024,
  "source_end": 3072
}
```

---

### 1.3 Model Entity

Tracks embedding models used for vector generation.

**Table**: `models`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | SERIAL | PRIMARY KEY | Unique model identifier |
| name | TEXT | UNIQUE NOT NULL | Model identifier (e.g., "sentence-transformers/all-MiniLM-L6-v2") |
| dimensions | INTEGER | NOT NULL | Embedding vector dimensions |
| provider | TEXT | | Model provider (e.g., "sentence-transformers", "openai") |
| description | TEXT | | Human-readable model description |
| created_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | First use timestamp |

**Relationships**:
- One model has many embeddings (1:N)

**Validation Rules**:
- `name` must be unique
- `dimensions` must be positive (typically 384, 768, 1536, etc.)
- `provider` should be from known set: "sentence-transformers", "openai", "anthropic"

**Standard Models**:
```sql
-- Default model for this demo
INSERT INTO models (name, dimensions, provider, description)
VALUES (
  'sentence-transformers/all-MiniLM-L6-v2',
  384,
  'sentence-transformers',
  'Fast and efficient 6-layer transformer for semantic similarity'
);
```

---

### 1.4 Embedding Entity

Stores vector embeddings for chunks.

**Table**: `embeddings`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | SERIAL | PRIMARY KEY | Unique embedding identifier |
| chunk_id | INTEGER | REFERENCES chunks(id) | Source chunk |
| model_id | INTEGER | REFERENCES models(id) | Embedding model used |
| embedding | VECTOR | NOT NULL | Vector embedding (dynamic dimensions) |
| created_at | TIMESTAMP WITH TIME ZONE | DEFAULT NOW() | Generation timestamp |

**Indexes**:
- UNIQUE(chunk_id, model_id) - one embedding per chunk per model
- Vector index for similarity search:
  ```sql
  CREATE INDEX embeddings_vector_idx ON embeddings
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);
  ```

**Relationships**:
- Many embeddings belong to one chunk (N:1)
- Many embeddings belong to one model (N:1)

**Validation Rules**:
- `embedding` dimensions must match `models.dimensions`
- Each (chunk_id, model_id) pair must be unique

---

## 2. LlamaIndex View (Denormalized Query Layer)

### 2.1 LlamaIndex Data View

PostgreSQL VIEW that adapts docs2db schema to LlamaIndex expectations.

**View**: `data_llamaindex`

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| id | BIGINT | embeddings.id | Unique record identifier |
| text | VARCHAR | chunks.text | Chunk text content |
| node_id | VARCHAR | Generated: 'chunk_' \|\| chunks.id | LlamaIndex node identifier |
| embedding | VECTOR(384) | embeddings.embedding | Vector embedding |
| metadata_ | JSONB | Constructed | Consolidated metadata from all joined tables |

**View Definition**:
```sql
CREATE OR REPLACE VIEW data_llamaindex AS
SELECT
    e.id::BIGINT as id,
    c.text::VARCHAR as text,
    ('chunk_' || c.id::TEXT)::VARCHAR as node_id,
    e.embedding as embedding,
    jsonb_build_object(
        'document_id', d.id,
        'document_path', d.path,
        'document_filename', d.filename,
        'content_type', d.content_type,
        'chunk_index', c.chunk_index,
        'chunk_id', c.id,
        'chunk_metadata', c.metadata,
        'model_id', m.id,
        'model_name', m.name,
        'model_dimensions', m.dimensions,
        'embedding_created_at', e.created_at
    )::JSONB as metadata_
FROM embeddings e
JOIN chunks c ON e.chunk_id = c.id
JOIN documents d ON c.document_id = d.id
JOIN models m ON e.model_id = m.id
WHERE m.name = 'sentence-transformers/all-MiniLM-L6-v2'
ORDER BY e.id;
```

**Metadata Structure**:
```json
{
  "document_id": 123,
  "document_path": "/docs/introduction.pdf",
  "document_filename": "introduction.pdf",
  "content_type": "application/pdf",
  "chunk_index": 5,
  "chunk_id": 456,
  "chunk_metadata": {
    "page_number": 3,
    "token_count": 512
  },
  "model_id": 1,
  "model_name": "sentence-transformers/all-MiniLM-L6-v2",
  "model_dimensions": 384,
  "embedding_created_at": "2025-10-21T10:30:00Z"
}
```

**View Characteristics**:
- **Read-only**: LlamaIndex cannot modify docs2db schema via view
- **Real-time**: Changes in docs2db immediately reflected in view
- **Filtered**: Shows only embeddings for specific model (all-MiniLM-L6-v2)
- **Performance**: Multi-table JOIN adds ~5-10ms latency

---

## 3. Query Entities (Runtime)

### 3.1 Query Entity

User's natural language question (not persisted).

| Field | Type | Description |
|-------|------|-------------|
| query_text | String | User's question |
| query_embedding | Vector(384) | Embedded query vector |
| top_k | Integer | Number of results to retrieve (default: 5) |
| similarity_threshold | Float | Minimum similarity score (default: 0.5) |
| timestamp | Timestamp | Query execution time |

**Example**:
```python
query = {
    "query_text": "What is pgvector?",
    "top_k": 5,
    "similarity_threshold": 0.5
}
```

---

### 3.2 Retrieved Context Entity

Set of chunks selected as relevant to query (not persisted).

| Field | Type | Description |
|-------|------|-------------|
| chunk_text | String | Retrieved chunk content |
| similarity_score | Float | Cosine similarity to query (0-1) |
| document_path | String | Source document path |
| document_filename | String | Source document name |
| chunk_index | Integer | Position in document |
| metadata | Object | Additional chunk metadata |

**Example**:
```python
context = [
    {
        "chunk_text": "pgvector is a PostgreSQL extension for vector similarity search...",
        "similarity_score": 0.87,
        "document_path": "/docs/pgvector_intro.pdf",
        "document_filename": "pgvector_intro.pdf",
        "chunk_index": 2,
        "metadata": {"page_number": 1}
    },
    # ... more chunks
]
```

---

### 3.3 Answer Entity

LLM-generated response (not persisted, optional logging).

| Field | Type | Description |
|-------|------|-------------|
| answer_text | String | Generated answer |
| source_citations | List[Object] | Source document references |
| confidence | Float | Confidence score (if available) |
| llm_provider | String | LLM used (e.g., "ollama", "openai") |
| llm_model | String | Model name (e.g., "llama3.1:latest") |
| generation_time | Float | Time to generate answer (seconds) |

**Example**:
```python
answer = {
    "answer_text": "pgvector is a PostgreSQL extension that enables...",
    "source_citations": [
        {"document": "pgvector_intro.pdf", "chunk_index": 2},
        {"document": "vector_search.md", "chunk_index": 0}
    ],
    "llm_provider": "ollama",
    "llm_model": "llama3.1:latest",
    "generation_time": 2.3
}
```

---

## 4. Configuration Entities

### 4.1 LLM Configuration

Provider-specific LLM configuration (from config.yaml).

**Structure**:
```yaml
llm:
  provider: "ollama"  # "ollama" | "openai" | "anthropic"

  ollama:
    model: "llama3.1:latest"
    request_timeout: 120.0
    context_window: 8000
    base_url: "http://localhost:11434"
    temperature: 0.7

  openai:
    model: "gpt-4o-mini"
    temperature: 0.7
    max_tokens: 512

  anthropic:
    model: "claude-sonnet-4-0"
    max_tokens: 1024
    temperature: 1.0
```

**Validation**:
- `provider` must be one of: "ollama", "openai", "anthropic"
- Provider-specific fields required when provider selected
- API keys loaded from environment variables

---

### 4.2 Embedding Configuration

Embedding model configuration (from config.yaml).

**Structure**:
```yaml
embedding:
  provider: "huggingface"  # "huggingface" | "ollama" | "openai"

  huggingface:
    model_name: "sentence-transformers/all-MiniLM-L6-v2"
    embed_dim: 384

  ollama:
    model_name: "nomic-embed-text"
    base_url: "http://localhost:11434"

  openai:
    model: "text-embedding-3-small"
    embed_dim: 1536
```

**Critical Constraint**:
- `embed_dim` MUST match database vector dimensions
- Changing embedding model requires recreating vector store

---

### 4.3 Retrieval Configuration

Retrieval parameters (from config.yaml).

**Structure**:
```yaml
retrieval:
  top_k: 5                      # Number of chunks to retrieve
  similarity_threshold: 0.5     # Minimum similarity score (0-1)
  chunk_size: 512              # Tokens per chunk
  chunk_overlap: 102           # Token overlap between chunks
```

---

## 5. Entity Relationships Diagram

```
┌─────────────┐
│  documents  │
│             │
│ • id        │◄───────┐
│ • path      │        │
│ • filename  │        │ 1:N
│ • metadata  │        │
└─────────────┘        │
                       │
                  ┌────┴────┐
                  │  chunks │
                  │         │
                  │ • id    │◄───────┐
                  │ • text  │        │
                  │ • index │        │ 1:N
                  │ • meta  │        │
                  └─────────┘        │
                                     │
┌──────────┐                   ┌─────┴────────┐
│  models  │                   │  embeddings  │
│          │                   │              │
│ • id     │◄──────────────────┤ • id         │
│ • name   │        1:N        │ • vector     │
│ • dims   │                   │ • chunk_id   │
└──────────┘                   │ • model_id   │
                               └──────────────┘
                                      │
                                      │ Exposed via VIEW
                                      │
                               ┌──────▼─────────────┐
                               │ data_llamaindex    │
                               │ (PostgreSQL VIEW)  │
                               │                    │
                               │ • id               │
                               │ • text             │
                               │ • node_id          │
                               │ • embedding        │
                               │ • metadata_        │
                               └────────────────────┘
                                      │
                                      │ Queried by
                                      │
                               ┌──────▼──────────┐
                               │  LlamaIndex     │
                               │  PGVectorStore  │
                               └─────────────────┘
```

---

## 6. Data Flow

### Ingestion Flow (docs2db)

```
1. User provides document folder
   ↓
2. docs2db processes documents
   ↓
3. Documents → `documents` table
   ↓
4. Text chunked (512 tokens, 20% overlap)
   ↓
5. Chunks → `chunks` table
   ↓
6. Embeddings generated (all-MiniLM-L6-v2)
   ↓
7. Embeddings → `embeddings` table
   ↓
8. Model info → `models` table
   ↓
9. Database dump created (ragdb_dump.sql)
```

### Query Flow (LlamaIndex)

```
1. User submits query text
   ↓
2. Query embedded using all-MiniLM-L6-v2
   ↓
3. Vector similarity search via `data_llamaindex` view
   ↓
4. Top-K chunks retrieved (with similarity scores)
   ↓
5. Retrieved context passed to LLM
   ↓
6. LLM generates answer with source citations
   ↓
7. Answer returned to user
```

---

## 7. Schema Validation

### Embedding Dimension Consistency

**Validation SQL**:
```sql
-- Verify all embeddings match expected dimensions
SELECT
    m.name,
    m.dimensions,
    COUNT(e.id) as embedding_count,
    vector_dims(e.embedding) as actual_dims
FROM embeddings e
JOIN models m ON e.model_id = m.id
GROUP BY m.name, m.dimensions, vector_dims(e.embedding)
HAVING m.dimensions != vector_dims(e.embedding);

-- Should return 0 rows if consistent
```

### View Integrity Check

**Validation SQL**:
```sql
-- Verify view returns expected structure
SELECT
    column_name,
    data_type
FROM information_schema.columns
WHERE table_name = 'data_llamaindex'
ORDER BY ordinal_position;

-- Expected output:
-- id          | bigint
-- text        | character varying
-- node_id     | character varying
-- embedding   | USER-DEFINED (vector)
-- metadata_   | jsonb
```

---

## 8. Data Constraints Summary

| Entity | Key Constraints |
|--------|-----------------|
| Document | Unique path, non-empty filename |
| Chunk | Unique (document_id, chunk_index), non-empty text |
| Model | Unique name, positive dimensions |
| Embedding | Unique (chunk_id, model_id), dimensions match model |
| View | Read-only, filtered to single model |

---

This data model enables efficient document ingestion via docs2db's normalized schema while providing optimized query performance through the denormalized LlamaIndex view layer.
