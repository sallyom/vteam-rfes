# Data Model: Llamaindex RAG Demo with Ramalama

**Feature**: 001-llamaindex-ramalama-demo
**Date**: 2025-10-31
**Source**: Derived from functional requirements and key entities in spec.md

## Overview

This document defines the data model for the RAG demo system. The model supports:
- Pre-populated vector database with docs2db-generated embeddings
- Query processing with retrieval and generation
- Result ranking and source attribution
- System health and operational monitoring

---

## Entity Definitions

### 1. Document Chunk

**Description**: A segment of documentation with associated vector embedding, stored in PGVector database.

**Attributes**:
- `id` (BIGSERIAL, PRIMARY KEY): Unique identifier for the chunk
- `node_id` (VARCHAR): Llamaindex node identifier
- `text` (TEXT, NOT NULL): Document content
- `embedding` (VECTOR(768)): Vector embedding from docs2db (Granite model)
- `metadata_` (JSONB): Structured metadata

**Metadata Fields** (JSONB):
- `source` (string): Original document source/file path
- `chunk_index` (integer): Position within source document
- `document_id` (string): Parent document identifier
- `title` (string, optional): Document or section title
- `category` (string, optional): Document category/type
- `created_at` (timestamp): Chunk creation time

**Relationships**:
- One document can have many chunks (one-to-many)
- Each chunk has one embedding vector (one-to-one)

**Validation Rules**:
- `text` must not be empty
- `embedding` dimension must be 768 (Granite model)
- `node_id` must be unique
- `metadata_` must be valid JSON

**Storage**:
- Table: `data_documents` (Llamaindex naming convention)
- Database: PostgreSQL with pgvector extension
- Index: HNSW on `embedding` column for fast similarity search
- Index: GIN on `metadata_` for metadata filtering

**Example**:
```json
{
  "id": 1,
  "node_id": "doc_001_chunk_001",
  "text": "Renewable energy sources include solar, wind, hydro, and geothermal power...",
  "embedding": [0.123, -0.456, 0.789, ...],  // 768-dimensional vector
  "metadata_": {
    "source": "docs/renewable_energy.md",
    "chunk_index": 0,
    "document_id": "doc_001",
    "title": "Introduction to Renewable Energy",
    "category": "technical",
    "created_at": "2025-10-30T10:00:00Z"
  }
}
```

---

### 2. RAG Query

**Description**: A user's natural language question submitted to the RAG system.

**Attributes**:
- `query_text` (string, required): User's natural language question
- `query_embedding` (vector(768)): Vector representation of the query
- `similarity_threshold` (float, optional): Minimum similarity score for results (default: 0.7)
- `max_chunks` (integer, optional): Maximum number of chunks to retrieve (default: 5)
- `metadata_filters` (object, optional): Filters to apply during retrieval
- `query_mode` (enum, optional): Retrieval strategy (default: "hybrid")
- `timestamp` (datetime): When query was submitted
- `query_id` (string): Unique identifier for tracking

**Query Modes**:
- `default`: Pure vector similarity search
- `sparse`: Text-based full-text search
- `hybrid`: Combined vector + text search

**Metadata Filters Structure**:
```json
{
  "filters": [
    {"key": "category", "value": "technical", "operator": "eq"},
    {"key": "created_at", "value": "2025-01-01", "operator": "gte"}
  ],
  "condition": "and"  // or "or"
}
```

**Validation Rules**:
- `query_text` must not be empty (min 1 character)
- `similarity_threshold` must be between 0.0 and 1.0
- `max_chunks` must be between 1 and 50
- `query_mode` must be one of: default, sparse, hybrid

**State Transitions**:
1. **Submitted**: Query received by RAG service
2. **Embedding**: Query converted to vector
3. **Retrieving**: Searching vector database
4. **Generating**: LLM generating response
5. **Complete**: Results returned to user
6. **Error**: Failed at any stage

**Example**:
```json
{
  "query_id": "query_abc123",
  "query_text": "What are the benefits of wind energy?",
  "query_embedding": [0.234, -0.567, ...],
  "similarity_threshold": 0.7,
  "max_chunks": 5,
  "metadata_filters": {
    "filters": [
      {"key": "category", "value": "technical", "operator": "eq"}
    ],
    "condition": "and"
  },
  "query_mode": "hybrid",
  "timestamp": "2025-10-31T14:30:00Z"
}
```

---

### 3. Query Result

**Description**: A retrieved document chunk with similarity score and metadata, part of the answer to a RAG query.

**Attributes**:
- `chunk_id` (integer): Reference to document chunk
- `chunk_text` (string): Retrieved document text
- `similarity_score` (float): Vector similarity score (0.0-1.0)
- `source_reference` (string): Source document path/identifier
- `chunk_metadata` (object): Metadata from the chunk
- `rank` (integer): Position in result set (1-based)

**Relationships**:
- Belongs to one RAG Query (many-to-one)
- References one Document Chunk (many-to-one)

**Validation Rules**:
- `similarity_score` must be between 0.0 and 1.0
- `chunk_text` must not be empty
- `rank` must be positive integer
- Results must be ordered by `similarity_score` descending

**Example**:
```json
{
  "chunk_id": 42,
  "chunk_text": "Wind energy is a clean, renewable power source that generates electricity...",
  "similarity_score": 0.87,
  "source_reference": "docs/renewable_energy.md",
  "chunk_metadata": {
    "title": "Wind Energy Benefits",
    "category": "technical",
    "section": "Chapter 3"
  },
  "rank": 1
}
```

---

### 4. RAG Response

**Description**: Complete answer generated by the RAG system, including synthesized response and source attributions.

**Attributes**:
- `query_id` (string): Reference to original query
- `answer_text` (string): LLM-generated synthesized answer
- `retrieved_chunks` (array): List of Query Results used
- `total_chunks_retrieved` (integer): Count of chunks retrieved
- `generation_tokens` (integer): Tokens used in LLM generation
- `latency_ms` (object): Timing breakdown
- `model_info` (object): Model details used
- `timestamp` (datetime): When response was generated

**Latency Breakdown**:
```json
{
  "embedding_ms": 50,
  "retrieval_ms": 120,
  "generation_ms": 800,
  "total_ms": 970
}
```

**Model Info**:
```json
{
  "llm_model": "Phi-4-mini-instruct-Q4_K_M",
  "embedding_model": "granite-embedding-30m-english",
  "llm_endpoint": "http://127.0.0.1:8080"
}
```

**Validation Rules**:
- `answer_text` must not be empty
- `retrieved_chunks` must contain at least 1 result
- `total_chunks_retrieved` must match length of `retrieved_chunks`
- `latency_ms.total_ms` should equal sum of component latencies

**Example**:
```json
{
  "query_id": "query_abc123",
  "answer_text": "Wind energy offers several benefits: it is renewable, produces no emissions, reduces dependence on fossil fuels...",
  "retrieved_chunks": [
    { "chunk_id": 42, "similarity_score": 0.87, ... },
    { "chunk_id": 15, "similarity_score": 0.82, ... },
    { "chunk_id": 78, "similarity_score": 0.79, ... }
  ],
  "total_chunks_retrieved": 3,
  "generation_tokens": 150,
  "latency_ms": {
    "embedding_ms": 50,
    "retrieval_ms": 120,
    "generation_ms": 800,
    "total_ms": 970
  },
  "model_info": {
    "llm_model": "Phi-4-mini-instruct-Q4_K_M",
    "embedding_model": "granite-embedding-30m-english",
    "llm_endpoint": "http://127.0.0.1:8080"
  },
  "timestamp": "2025-10-31T14:30:01Z"
}
```

---

### 5. Model Server Configuration

**Description**: Configuration and runtime state of the Ramalama model serving component.

**Attributes**:
- `model_name` (string): Identifier of the loaded model
- `model_path` (string): File path to model artifacts
- `endpoint_url` (string): HTTP endpoint for API access
- `port` (integer): Port number for API
- `runtime` (enum): Inference backend (llama.cpp, vLLM, MLX)
- `context_size` (integer): Maximum context window size
- `gpu_layers` (integer): Number of layers offloaded to GPU (-1=auto, 0=CPU only)
- `health_status` (enum): Current health state

**Runtime Options**:
- `llama.cpp`: Default, broad compatibility
- `vLLM`: High-throughput production
- `MLX`: Apple Silicon optimization

**Health Status**:
- `initializing`: Model loading in progress
- `healthy`: Ready to accept requests
- `degraded`: Operational but with issues
- `unhealthy`: Not accepting requests

**Validation Rules**:
- `port` must be between 1024 and 65535
- `context_size` must be power of 2 (e.g., 2048, 4096, 8192)
- `gpu_layers` must be >= -1
- `endpoint_url` must be valid HTTP/HTTPS URL

**Example**:
```json
{
  "model_name": "Phi-4-mini-instruct-Q4_K_M",
  "model_path": "/var/lib/ramalama/models/unsloth/Phi-4-mini-instruct-GGUF/Phi-4-mini-instruct-Q4_K_M.gguf",
  "endpoint_url": "http://127.0.0.1:8080",
  "port": 8080,
  "runtime": "llama.cpp",
  "context_size": 4096,
  "gpu_layers": -1,
  "health_status": "healthy"
}
```

---

### 6. Vector Database Configuration

**Description**: Configuration for PostgreSQL + PGVector database connection and indexing.

**Attributes**:
- `host` (string): Database server hostname
- `port` (integer): Database server port
- `database_name` (string): Database name
- `table_name` (string): Vector store table name
- `schema_name` (string): Database schema (default: "public")
- `embedding_dimension` (integer): Vector dimension
- `index_type` (enum): Indexing method (hnsw, ivfflat)
- `index_params` (object): Index-specific parameters
- `connection_pool_size` (integer): Connection pool size
- `health_status` (enum): Database health state

**Index Types**:
- `hnsw`: Hierarchical Navigable Small World (default)
- `ivfflat`: Inverted File with Flat Index

**HNSW Index Parameters**:
```json
{
  "hnsw_m": 16,
  "hnsw_ef_construction": 64,
  "hnsw_ef_search": 40,
  "hnsw_dist_method": "vector_cosine_ops"
}
```

**Health Status**:
- `connected`: Database accessible and responding
- `disconnected`: Cannot connect to database
- `readonly`: Connected but read-only mode

**Validation Rules**:
- `port` must be between 1024 and 65535
- `embedding_dimension` must match model output (typically 768)
- `connection_pool_size` must be between 1 and 100
- `index_type` must be one of: hnsw, ivfflat

**Example**:
```json
{
  "host": "127.0.0.1",
  "port": 5432,
  "database_name": "ragdb",
  "table_name": "documents",
  "schema_name": "public",
  "embedding_dimension": 768,
  "index_type": "hnsw",
  "index_params": {
    "hnsw_m": 16,
    "hnsw_ef_construction": 64,
    "hnsw_ef_search": 40,
    "hnsw_dist_method": "vector_cosine_ops"
  },
  "connection_pool_size": 10,
  "health_status": "connected"
}
```

---

### 7. Container Configuration

**Description**: Deployment configuration for containerized services in the pod.

**Attributes**:
- `container_name` (string): Container identifier
- `image` (string): Container image reference
- `ports` (array): Port mappings
- `environment_vars` (object): Environment variable overrides
- `volume_mounts` (array): Volume mount specifications
- `resources` (object): Resource requests and limits
- `health_check` (object): Health probe configuration
- `status` (enum): Current container state

**Port Mapping**:
```json
{
  "container_port": 5432,
  "host_port": 5432,
  "protocol": "tcp"
}
```

**Resource Specification**:
```json
{
  "requests": {
    "memory": "1Gi",
    "cpu": "500m"
  },
  "limits": {
    "memory": "2Gi",
    "cpu": "1000m"
  }
}
```

**Health Check**:
```json
{
  "type": "http",
  "path": "/health",
  "port": 8080,
  "initial_delay_seconds": 10,
  "period_seconds": 30,
  "timeout_seconds": 5,
  "failure_threshold": 3
}
```

**Container Status**:
- `creating`: Container being created
- `running`: Container operational
- `restarting`: Container restarting due to failure
- `stopped`: Container intentionally stopped
- `failed`: Container crashed

**Example**:
```json
{
  "container_name": "rag-service",
  "image": "localhost/llamaindex-rag:latest",
  "ports": [
    {"container_port": 8000, "host_port": 8000, "protocol": "tcp"}
  ],
  "environment_vars": {
    "POSTGRES_HOST": "127.0.0.1",
    "RAMALAMA_URL": "http://127.0.0.1:8080",
    "LOG_LEVEL": "INFO"
  },
  "volume_mounts": [
    {"name": "config", "mount_path": "/app/config", "readonly": true}
  ],
  "resources": {
    "requests": {"memory": "1Gi", "cpu": "500m"},
    "limits": {"memory": "2Gi", "cpu": "1000m"}
  },
  "health_check": {
    "type": "http",
    "path": "/health",
    "port": 8000,
    "initial_delay_seconds": 5,
    "period_seconds": 10,
    "timeout_seconds": 5,
    "failure_threshold": 3
  },
  "status": "running"
}
```

---

## Entity Relationships

```
Document Chunk (1) --- (*) Query Result
Query Result (*) --- (1) RAG Query
RAG Query (1) --- (1) RAG Response
RAG Response (*) --- (*) Query Result

Model Server Configuration (1) --- (*) RAG Query (via embedding generation)
Vector Database Configuration (1) --- (*) Query Result (via retrieval)
Container Configuration (*) --- (1) Pod Deployment
```

**Key Relationships**:
1. One **RAG Query** retrieves many **Query Results**
2. One **RAG Response** includes many **Query Results**
3. Each **Query Result** references one **Document Chunk**
4. **Model Server** generates embeddings for all queries
5. **Vector Database** stores all document chunks
6. **Container Configuration** defines each service in the pod

---

## Database Schema

### PGVector Schema (PostgreSQL)

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;  -- For full-text search

-- Main document storage table
CREATE TABLE data_documents (
    id BIGSERIAL PRIMARY KEY,
    node_id VARCHAR(255) UNIQUE NOT NULL,
    text TEXT NOT NULL,
    embedding VECTOR(768) NOT NULL,
    metadata_ JSONB,
    text_search_tsv TSVECTOR,  -- For hybrid search
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- HNSW index for fast vector similarity search
CREATE INDEX idx_documents_embedding ON data_documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- GIN index for full-text search
CREATE INDEX idx_documents_text_search ON data_documents
USING GIN (text_search_tsv);

-- GIN index for metadata queries
CREATE INDEX idx_documents_metadata ON data_documents
USING GIN (metadata_);

-- Index for node_id lookups
CREATE INDEX idx_documents_node_id ON data_documents(node_id);

-- Function to update text search vector
CREATE OR REPLACE FUNCTION update_text_search_tsv()
RETURNS TRIGGER AS $$
BEGIN
    NEW.text_search_tsv := to_tsvector('english', NEW.text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger to maintain text search vector
CREATE TRIGGER trig_update_text_search_tsv
    BEFORE INSERT OR UPDATE ON data_documents
    FOR EACH ROW
    EXECUTE FUNCTION update_text_search_tsv();

-- View for document statistics
CREATE VIEW document_stats AS
SELECT
    COUNT(*) as total_chunks,
    COUNT(DISTINCT metadata_->>'document_id') as total_documents,
    AVG(char_length(text)) as avg_chunk_length,
    jsonb_object_agg(
        metadata_->>'category',
        COUNT(*)
    ) FILTER (WHERE metadata_->>'category' IS NOT NULL) as chunks_by_category
FROM data_documents;
```

### Indexes Rationale

1. **HNSW on embedding**: Fast approximate nearest neighbor search (< 200ms for most queries)
2. **GIN on text_search_tsv**: Enables hybrid search with full-text capabilities
3. **GIN on metadata_**: Efficient filtering on JSON metadata fields
4. **B-tree on node_id**: Fast lookups by Llamaindex node identifiers

---

## Validation Rules Summary

### Document Chunk
- ✓ text: not empty
- ✓ embedding: 768 dimensions
- ✓ node_id: unique across table
- ✓ metadata_: valid JSON

### RAG Query
- ✓ query_text: min 1 character
- ✓ similarity_threshold: 0.0-1.0
- ✓ max_chunks: 1-50
- ✓ query_mode: default | sparse | hybrid

### Query Result
- ✓ similarity_score: 0.0-1.0
- ✓ chunk_text: not empty
- ✓ rank: positive integer
- ✓ ordered by score descending

### Model Server Configuration
- ✓ port: 1024-65535
- ✓ context_size: power of 2
- ✓ gpu_layers: >= -1
- ✓ endpoint_url: valid URL

### Vector Database Configuration
- ✓ port: 1024-65535
- ✓ embedding_dimension: matches model (768)
- ✓ connection_pool_size: 1-100
- ✓ index_type: hnsw | ivfflat

---

## Performance Considerations

### Vector Search Optimization
- **HNSW parameters**: Tuned for <200ms p95 latency
  - `m=16`: Balance between recall and memory
  - `ef_construction=64`: Build quality vs build time
  - `ef_search=40`: Query speed vs recall
- **Expected recall**: >95% with these settings

### Connection Pooling
- **Pool size**: 10 connections (handles 10 concurrent queries)
- **Max overflow**: 20 additional connections under load
- **Pool timeout**: 30 seconds before connection error

### Query Performance Targets
- **Vector search**: <200ms p95
- **Full query (including LLM)**: <5 seconds p95
- **Concurrent queries**: 10 without degradation

### Storage Estimates
- **Vector size**: 768 floats × 4 bytes = 3KB per chunk
- **Text size**: ~500 bytes average per chunk
- **Metadata size**: ~200 bytes per chunk
- **Total per chunk**: ~3.7KB
- **10K documents (~50K chunks)**: ~185MB database size

---

## Next Steps

With the data model defined, proceed to:
1. **API contracts**: Define REST endpoints for query service
2. **Quickstart guide**: Document deployment and testing
3. **Agent context update**: Add technologies to agent knowledge base
