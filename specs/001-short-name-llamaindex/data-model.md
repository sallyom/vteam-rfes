# Data Model: Llamaindex + RamaLama RAG Demo

**Feature**: [Llamaindex + RamaLama RAG Demo](./spec.md)
**Date**: 2025-10-30
**Status**: Design Complete

## Overview

This document defines the data entities for the RAG demo system. The demo integrates with an existing docs2db database schema and adds RAG-specific entities for query processing and response generation.

---

## Entity Definitions

### 1. Document (Existing - docs2db schema)

**Purpose**: Represents source files processed into the system

**Attributes**:
- `id` (INTEGER, PRIMARY KEY): Unique document identifier
- `source_file` (TEXT, NOT NULL): Original file path/name
- `document_type` (TEXT): File format (pdf, html, markdown, txt)
- `processing_timestamp` (TIMESTAMP): When document was processed
- `status` (TEXT): Processing status (completed, failed, pending)
- `metadata` (JSONB): Additional document properties (author, title, date)
- `hash` (TEXT): Content hash for deduplication

**Relationships**:
- One-to-many with Chunk (one document has many chunks)

**Validation Rules**:
- `source_file` must be unique
- `status` must be one of: completed, failed, pending
- `processing_timestamp` defaults to NOW()

**State Transitions**:
- pending → processing → completed
- processing → failed (on error)

---

### 2. Chunk (Existing - docs2db schema)

**Purpose**: Segmented portions of documents optimized for retrieval

**Attributes**:
- `id` (INTEGER, PRIMARY KEY): Unique chunk identifier
- `document_id` (INTEGER, FOREIGN KEY → documents.id): Parent document reference
- `chunk_text` (TEXT, NOT NULL): Content text for embedding and retrieval
- `chunk_index` (INTEGER): Position within parent document (0-based)
- `token_count` (INTEGER): Number of tokens in chunk
- `metadata` (JSONB): Chunking metadata (strategy, overlap, boundaries)

**Relationships**:
- Many-to-one with Document (many chunks per document)
- One-to-many with Embedding (one chunk can have embeddings from multiple models)

**Validation Rules**:
- `chunk_text` must not be empty
- `chunk_index` >= 0
- `token_count` > 0
- Unique constraint on (document_id, chunk_index)

**Indexing**:
- Index on `document_id` for retrieval
- Full-text search index on `chunk_text` for hybrid search

---

### 3. Model (Existing - docs2db schema)

**Purpose**: Embedding model configuration and metadata

**Attributes**:
- `id` (INTEGER, PRIMARY KEY): Unique model identifier
- `name` (TEXT, UNIQUE, NOT NULL): Full model identifier (e.g., "ibm-granite/granite-embedding-30m-english")
- `dimensions` (INTEGER, NOT NULL): Embedding vector dimensions
- `provider` (TEXT): Model provider (sentence-transformers, openai, etc.)
- `description` (TEXT): Model details and use case
- `created_at` (TIMESTAMP): When model was registered

**Relationships**:
- One-to-many with Embedding (one model produces many embeddings)

**Validation Rules**:
- `name` must be unique
- `dimensions` must be > 0 and match actual model output
- Common dimensions: 384, 768, 1024, 1536

**Critical Note**: System validates that query-time embedding model matches database model. Mismatch causes startup failure.

---

### 4. Embedding (Existing - docs2db schema)

**Purpose**: Vector representations of chunk content for semantic search

**Attributes**:
- `id` (INTEGER, PRIMARY KEY): Unique embedding identifier
- `chunk_id` (INTEGER, FOREIGN KEY → chunks.id): Source chunk reference
- `model_id` (INTEGER, FOREIGN KEY → models.id): Model used for generation
- `embedding` (VECTOR): Vector representation (pgvector type)
- `generated_at` (TIMESTAMP): When embedding was created

**Relationships**:
- Many-to-one with Chunk (many embeddings per chunk if using multiple models)
- Many-to-one with Model (many embeddings from same model)

**Validation Rules**:
- `embedding` dimensions must match `models.dimensions`
- Unique constraint on (chunk_id, model_id)

**Indexing**:
- **HNSW index** on `embedding` for fast similarity search:
  ```sql
  CREATE INDEX embeddings_hnsw_idx ON embeddings
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);
  ```

**Performance**:
- Without index: O(n) scan, 2-5 seconds for 10K vectors
- With HNSW: O(log n) search, 50-200ms for 10K vectors

---

### 5. Query (New - RAG demo entity)

**Purpose**: User information requests to the RAG system

**Attributes**:
- `id` (UUID, PRIMARY KEY): Unique query identifier
- `query_text` (TEXT, NOT NULL): Natural language query
- `timestamp` (TIMESTAMP, NOT NULL): When query was submitted
- `retrieval_params` (JSONB): Query parameters
  - `top_k` (INTEGER): Number of chunks to retrieve (default: 5)
  - `similarity_threshold` (FLOAT): Minimum similarity score (default: 0.7)
  - `enable_hybrid` (BOOLEAN): Use hybrid search (default: false)
- `user_id` (TEXT, NULLABLE): User identifier (for multi-user scenarios)
- `session_id` (TEXT, NULLABLE): Session context identifier

**Relationships**:
- One-to-one with Response (one query produces one response)
- Many-to-many with Chunk (through RetrievedChunk join table)

**Validation Rules**:
- `query_text` must not be empty (min 3 characters)
- `retrieval_params.top_k`: 1 <= value <= 20
- `retrieval_params.similarity_threshold`: 0.0 <= value <= 1.0

**Usage Pattern**:
- Created when user submits query
- Persisted for analytics and debugging
- Optional: Can be used for query caching

---

### 6. Response (New - RAG demo entity)

**Purpose**: System-generated answers to queries

**Attributes**:
- `id` (UUID, PRIMARY KEY): Unique response identifier
- `query_id` (UUID, FOREIGN KEY → queries.id): Associated query
- `answer_text` (TEXT, NOT NULL): Generated response with citations
- `processing_time_ms` (INTEGER): End-to-end latency in milliseconds
- `llm_model` (TEXT): LLM used for generation (e.g., "tinyllama-1.1b")
- `confidence` (TEXT): Confidence level (high, medium, low, none)
- `timestamp` (TIMESTAMP, NOT NULL): When response was generated
- `metadata` (JSONB): Additional response information
  - `tokens_generated` (INTEGER): Number of tokens in response
  - `retrieval_time_ms` (INTEGER): Time spent on retrieval
  - `generation_time_ms` (INTEGER): Time spent on LLM generation
  - `chunks_retrieved` (INTEGER): Number of chunks retrieved

**Relationships**:
- Many-to-one with Query (one response per query)
- Many-to-many with Chunk (through RetrievedChunk, indicating sources)

**Validation Rules**:
- `answer_text` must not be empty
- `confidence` must be one of: high, medium, low, none
- `processing_time_ms` > 0

**Confidence Calculation**:
- **high**: Top chunk similarity >= 0.8
- **medium**: Top chunk similarity 0.7-0.8
- **low**: Top chunk similarity 0.6-0.7
- **none**: No chunks above threshold

---

### 7. RetrievedChunk (New - RAG demo join table)

**Purpose**: Links queries/responses to source chunks with relevance scores

**Attributes**:
- `query_id` (UUID, FOREIGN KEY → queries.id): Query reference
- `chunk_id` (INTEGER, FOREIGN KEY → chunks.id): Chunk reference
- `similarity_score` (FLOAT, NOT NULL): Relevance score (0.0-1.0)
- `rank` (INTEGER): Retrieval ranking (1 = most relevant)
- `used_in_response` (BOOLEAN): Whether chunk was included in context

**Relationships**:
- Many-to-one with Query
- Many-to-one with Chunk

**Validation Rules**:
- `similarity_score`: 0.0 <= value <= 1.0
- `rank` >= 1
- Unique constraint on (query_id, chunk_id)

**Usage**:
- Enables source citation in responses
- Provides observability into retrieval quality
- Supports debugging and analytics

---

## Schema Diagram

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│  Document   │────────>│    Chunk    │────────>│  Embedding  │
│  (existing) │  1:N    │  (existing) │  N:1    │  (existing) │
└─────────────┘         └─────────────┘         └─────────────┘
                              │                         │
                              │                    ┌────┴────┐
                              │                    │  Model  │
                              │                    │(existing)│
                              │                    └─────────┘
                              │
                        ┌─────┴──────┐
                        │ Retrieved  │
                        │   Chunk    │
                        │   (new)    │
                        └─────┬──────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
              ┌─────▼────┐        ┌─────▼────┐
              │  Query   │───────>│ Response │
              │  (new)   │  1:1   │  (new)   │
              └──────────┘        └──────────┘
```

---

## Database Schema (SQL)

### Existing docs2db Tables (reference)

```sql
-- Already exists in docs2db
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    source_file TEXT UNIQUE NOT NULL,
    document_type TEXT,
    processing_timestamp TIMESTAMP DEFAULT NOW(),
    status TEXT DEFAULT 'pending',
    metadata JSONB,
    hash TEXT
);

CREATE TABLE chunks (
    id SERIAL PRIMARY KEY,
    document_id INTEGER REFERENCES documents(id) ON DELETE CASCADE,
    chunk_text TEXT NOT NULL,
    chunk_index INTEGER NOT NULL,
    token_count INTEGER,
    metadata JSONB,
    UNIQUE(document_id, chunk_index)
);

CREATE TABLE models (
    id SERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,
    dimensions INTEGER NOT NULL,
    provider TEXT,
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE embeddings (
    id SERIAL PRIMARY KEY,
    chunk_id INTEGER REFERENCES chunks(id) ON DELETE CASCADE,
    model_id INTEGER REFERENCES models(id),
    embedding VECTOR,
    generated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(chunk_id, model_id)
);

-- Performance index (create post-load)
CREATE INDEX embeddings_hnsw_idx ON embeddings
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Hybrid search support
CREATE INDEX chunks_text_idx ON chunks
USING gin(to_tsvector('english', chunk_text));
```

### New RAG Demo Tables

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE queries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    query_text TEXT NOT NULL CHECK (length(query_text) >= 3),
    timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
    retrieval_params JSONB DEFAULT '{"top_k": 5, "similarity_threshold": 0.7, "enable_hybrid": false}'::jsonb,
    user_id TEXT,
    session_id TEXT
);

CREATE TABLE responses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    query_id UUID REFERENCES queries(id) ON DELETE CASCADE,
    answer_text TEXT NOT NULL,
    processing_time_ms INTEGER NOT NULL CHECK (processing_time_ms > 0),
    llm_model TEXT NOT NULL,
    confidence TEXT CHECK (confidence IN ('high', 'medium', 'low', 'none')),
    timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
    metadata JSONB
);

CREATE TABLE retrieved_chunks (
    query_id UUID REFERENCES queries(id) ON DELETE CASCADE,
    chunk_id INTEGER REFERENCES chunks(id) ON DELETE CASCADE,
    similarity_score FLOAT NOT NULL CHECK (similarity_score >= 0.0 AND similarity_score <= 1.0),
    rank INTEGER NOT NULL CHECK (rank >= 1),
    used_in_response BOOLEAN DEFAULT true,
    PRIMARY KEY (query_id, chunk_id)
);

-- Indexes for query performance
CREATE INDEX queries_timestamp_idx ON queries(timestamp DESC);
CREATE INDEX responses_query_id_idx ON responses(query_id);
CREATE INDEX retrieved_chunks_query_id_idx ON retrieved_chunks(query_id);
CREATE INDEX retrieved_chunks_chunk_id_idx ON retrieved_chunks(chunk_id);
```

---

## Data Flow

### Query Processing Flow

1. **Query Submission**:
   - User submits `query_text` with optional `retrieval_params`
   - System creates `Query` entity with timestamp

2. **Embedding Generation**:
   - Query text embedded using same model as document embeddings
   - Model validation ensures consistency (startup check)

3. **Vector Similarity Search**:
   - Query embedding compared against `embeddings` table
   - HNSW index used for fast approximate nearest neighbor search
   - Results ranked by cosine similarity

4. **Chunk Retrieval**:
   - Top-k chunks retrieved based on similarity scores
   - Chunks with score < `similarity_threshold` filtered out
   - `RetrievedChunk` records created for observability

5. **Context Assembly**:
   - Retrieved chunks assembled into context
   - Token budget enforced (3000 tokens for TinyLlama)
   - Chunks prioritized by similarity score

6. **LLM Generation**:
   - Context + query sent to RamaLama LLM
   - Response generated with source citations
   - `Response` entity created with answer and metadata

7. **Result Return**:
   - Response returned to user with sources
   - Performance metrics logged (retrieval_time, generation_time)

---

## Storage Considerations

### Data Volumes (Demo Scale)

| Entity | Est. Count | Storage Per Row | Total Storage |
|--------|------------|----------------|---------------|
| Documents | 100-1000 | ~5 KB | 0.5-5 MB |
| Chunks | 5K-50K | ~1 KB | 5-50 MB |
| Embeddings | 5K-50K | ~1.5 KB (384d) | 7.5-75 MB |
| Queries | 100-1K | ~0.5 KB | 50-500 KB |
| Responses | 100-1K | ~2 KB | 200 KB-2 MB |
| RetrievedChunks | 500-5K | ~0.1 KB | 50-500 KB |

**Total Storage**: ~15-135 MB (well within 20GB constraint)

### Index Storage

- HNSW index: ~2x embedding storage = 15-150 MB
- Full-text index: ~1x chunk text = 5-50 MB

**Total with Indexes**: ~35-335 MB

---

## Performance Characteristics

### Query Latency Budget (Target: <5s)

| Operation | Budget | Notes |
|-----------|--------|-------|
| Embedding generation | 100-300ms | CPU-based sentence-transformers |
| Vector search (HNSW) | 50-200ms | 10K vectors |
| Context assembly | 50-100ms | Python processing |
| LLM generation (100 tokens) | 2-4s | TinyLlama CPU inference |
| **Total** | **2.5-5s** | Meets requirement |

### Scaling Characteristics

- **Read-heavy**: 95% reads (queries), 5% writes (ingestion)
- **Horizontal scaling**: Connection pooling for concurrent queries
- **Bottleneck**: LLM generation (sequential, CPU-bound)

---

## Data Retention Policy

**Demo Scope** (not production):
- Keep all queries/responses for session duration
- Optional: Implement query history cleanup (>1000 queries)
- Production would need: retention policy, archival, GDPR compliance

---

## Testing Validation

### Entity Tests

- [x] All entities have primary keys
- [x] Foreign key relationships defined
- [x] Validation constraints specified
- [x] Indexes for query performance
- [x] No circular dependencies

### Integration Tests

- Test embedding model validation (mismatch detection)
- Test query processing end-to-end
- Test similarity threshold filtering
- Test citation tracking through RetrievedChunk
- Test performance within latency budget

---

**Data Model Status**: Complete and ready for contract generation.
