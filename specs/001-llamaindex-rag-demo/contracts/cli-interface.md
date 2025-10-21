# CLI Interface Contract

**Feature**: LlamaIndex + pgvectordb RAG Demo
**Version**: 1.0.0
**Date**: 2025-10-21

This document specifies the command-line interface contracts for the RAG demo.

---

## 1. Document Ingestion CLI

### Command: `ingest_documents.py`

Ingest documents into docs2db and create pgvector database.

**Usage**:
```bash
python ingestion/ingest_documents.py \
  --input-dir <path> \
  --output-db <connection-string> \
  [--chunk-size 512] \
  [--chunk-overlap 102] \
  [--embedding-model all-MiniLM-L6-v2] \
  [--verbose]
```

**Parameters**:
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `--input-dir` | Path | Yes | - | Directory containing documents to ingest |
| `--output-db` | String | Yes | - | PostgreSQL connection string (postgresql://user:pass@host:port/db) |
| `--chunk-size` | Integer | No | 512 | Token count per chunk |
| `--chunk-overlap` | Integer | No | 102 | Token overlap between chunks |
| `--embedding-model` | String | No | all-MiniLM-L6-v2 | Embedding model name |
| `--verbose` | Flag | No | False | Enable verbose logging |

**Exit Codes**:
- `0`: Success - all documents ingested
- `1`: Invalid arguments or configuration error
- `2`: Database connection error
- `3`: Embedding model not found
- `4`: Document processing error

**Output** (stdout):
```json
{
  "status": "success",
  "documents_processed": 15,
  "chunks_created": 1243,
  "embeddings_generated": 1243,
  "database_size_mb": 125.4,
  "processing_time_seconds": 45.2
}
```

**Example**:
```bash
python ingestion/ingest_documents.py \
  --input-dir ./examples/sample_docs \
  --output-db postgresql://postgres:password@localhost:5432/ragdb \
  --verbose
```

---

## 2. Database Dump Creation CLI

### Command: `create_dump.sh`

Create PostgreSQL dump file from ingested database.

**Usage**:
```bash
./ingestion/create_dump.sh \
  <database-name> \
  <output-file> \
  [postgres-host] \
  [postgres-port] \
  [postgres-user]
```

**Parameters**:
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `database-name` | String | Yes | - | PostgreSQL database name |
| `output-file` | Path | Yes | - | Output dump file path (e.g., ragdb_dump.sql) |
| `postgres-host` | String | No | localhost | PostgreSQL host |
| `postgres-port` | Integer | No | 5432 | PostgreSQL port |
| `postgres-user` | String | No | postgres | PostgreSQL username |

**Environment Variables**:
- `POSTGRES_PASSWORD`: Database password (required)

**Exit Codes**:
- `0`: Success - dump created
- `1`: Invalid arguments
- `2`: Database connection error
- `3`: pg_dump command failed
- `4`: Insufficient disk space

**Output** (stdout):
```
Creating database dump...
Dumping schema...
Dumping data...
Dump created: ragdb_dump.sql (125.4 MB)
```

**Example**:
```bash
export POSTGRES_PASSWORD=mypassword
./ingestion/create_dump.sh ragdb ragdb_dump.sql localhost 5432 postgres
```

---

## 3. Query Execution CLI

### Command: `query_cli.py`

Execute queries against the RAG system.

**Usage**:
```bash
python llamaindex-service/query_cli.py \
  --query "<question>" \
  [--config config.yaml] \
  [--top-k 5] \
  [--similarity-threshold 0.5] \
  [--explain] \
  [--json]
```

**Parameters**:
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `--query` | String | Yes* | - | Query text (mutually exclusive with --interactive) |
| `--config` | Path | No | config.yaml | Configuration file path |
| `--top-k` | Integer | No | 5 | Number of chunks to retrieve |
| `--similarity-threshold` | Float | No | 0.5 | Minimum similarity score (0-1) |
| `--explain` | Flag | No | False | Show detailed retrieval information |
| `--json` | Flag | No | False | Output in JSON format |
| `--interactive` | Flag | No | False | Enter interactive chat mode |

**Exit Codes**:
- `0`: Success - query executed
- `1`: Invalid arguments or configuration error
- `2`: Database connection error
- `3`: LLM service unavailable
- `4`: No relevant results found
- `5`: Embedding model mismatch

**Output** (human-readable, default):
```
Query: What is pgvector?

Answer:
pgvector is a PostgreSQL extension that enables vector similarity search directly
within PostgreSQL databases. It provides vector data types and indexes optimized
for semantic search applications.

Sources:
[1] pgvector_intro.pdf (page 1, similarity: 0.87)
[2] vector_databases.md (chunk 3, similarity: 0.82)
[3] postgres_extensions.pdf (page 5, similarity: 0.76)

Query time: 2.3s
```

**Output** (JSON format, `--json` flag):
```json
{
  "query": "What is pgvector?",
  "answer": "pgvector is a PostgreSQL extension...",
  "sources": [
    {
      "document": "pgvector_intro.pdf",
      "chunk_index": 2,
      "similarity_score": 0.87,
      "metadata": {"page_number": 1}
    }
  ],
  "llm_provider": "ollama",
  "llm_model": "llama3.1:latest",
  "query_time_seconds": 2.3,
  "retrieval_count": 5,
  "embedding_model": "sentence-transformers/all-MiniLM-L6-v2"
}
```

**Output** (explain mode, `--explain` flag):
```
Query: What is pgvector?

Step 1: Query Embedding
  Model: sentence-transformers/all-MiniLM-L6-v2
  Dimensions: 384
  Embedding time: 0.1s

Step 2: Vector Similarity Search
  Database: ragdb (localhost:5432)
  Top-K: 5
  Similarity threshold: 0.5
  Search time: 0.2s

Retrieved Chunks:
  [1] Similarity: 0.87 | pgvector_intro.pdf (chunk 2)
      Text: "pgvector is a PostgreSQL extension that..."
  [2] Similarity: 0.82 | vector_databases.md (chunk 3)
      Text: "Vector databases like pgvector provide..."
  [3] Similarity: 0.76 | postgres_extensions.pdf (chunk 15)
      Text: "Popular PostgreSQL extensions include pgvector..."

Step 3: LLM Generation
  Provider: ollama
  Model: llama3.1:latest
  Context length: 2048 tokens
  Temperature: 0.7
  Generation time: 2.0s

Answer:
pgvector is a PostgreSQL extension that enables...

Total time: 2.3s
```

**Example**:
```bash
# Simple query
python llamaindex-service/query_cli.py \
  --query "What is pgvector?"

# Query with custom parameters
python llamaindex-service/query_cli.py \
  --query "How does vector search work?" \
  --top-k 10 \
  --similarity-threshold 0.7 \
  --explain

# Interactive mode
python llamaindex-service/query_cli.py --interactive
```

---

## 4. Interactive Chat Mode

### Command: `query_cli.py --interactive`

Enter interactive chat session with conversation history.

**Usage**:
```bash
python llamaindex-service/query_cli.py --interactive [--config config.yaml]
```

**Interactive Commands**:
| Command | Description |
|---------|-------------|
| `<query text>` | Submit query and get answer |
| `/explain` | Show detailed information about last query |
| `/sources` | Show sources from last query |
| `/config` | Display current configuration |
| `/history` | Show conversation history |
| `/clear` | Clear conversation history |
| `/help` | Show help information |
| `/exit` or `Ctrl+D` | Exit interactive mode |

**Example Session**:
```
RAG Interactive Chat
Type /help for commands, /exit to quit
Config: config.yaml | LLM: ollama/llama3.1:latest | DB: ragdb

> What is pgvector?

pgvector is a PostgreSQL extension that enables vector similarity search...

Sources: [1] pgvector_intro.pdf [2] vector_databases.md
Query time: 2.3s

> /explain

[Shows detailed explanation of last query]

> How do I install it?

To install pgvector, you can use the following methods...

> /exit

Goodbye!
```

---

## 5. Health Check CLI

### Command: `health_check.py`

Verify system components are healthy and ready.

**Usage**:
```bash
python llamaindex-service/health_check.py [--config config.yaml] [--verbose]
```

**Exit Codes**:
- `0`: All checks passed
- `1`: One or more checks failed

**Output**:
```
RAG System Health Check
=======================

[✓] Configuration file: config.yaml
[✓] Database connection: ragdb@localhost:5432
[✓] Database view: data_llamaindex (1243 rows)
[✓] Embedding model: sentence-transformers/all-MiniLM-L6-v2 (384 dims)
[✓] LLM service: ollama (llama3.1:latest)
[✓] Vector dimensions: 384 (matches embedding model)

All checks passed!
```

**Failure Example**:
```
RAG System Health Check
=======================

[✓] Configuration file: config.yaml
[✗] Database connection: Connection refused (localhost:5432)
[!] Database view: Skipped (database unreachable)
[✓] Embedding model: sentence-transformers/all-MiniLM-L6-v2 (384 dims)
[✗] LLM service: Ollama not running (http://localhost:11434)
[!] Vector dimensions: Skipped (database unreachable)

2 checks failed, 1 warning
```

---

## 6. Container Deployment CLI

### Command: `podman play kube` or `docker-compose`

Deploy containers using pod.yaml specification.

**Usage (Podman)**:
```bash
podman play kube pod.yaml
```

**Usage (Docker Compose equivalent)**:
```bash
docker-compose up -d
```

**Container Verification**:
```bash
# Check container status
podman pod ps
podman ps --pod

# Check logs
podman logs <pgvector-container-name>
podman logs <llamaindex-container-name>

# Execute commands in containers
podman exec -it <pgvector-container-name> psql -U postgres -d ragdb
podman exec -it <llamaindex-container-name> python query_cli.py --query "test"
```

**Expected Output** (successful deployment):
```
Pod:
NAME                   STATUS      AGE
rag-demo-pod           Running     2m

Containers:
CONTAINER ID  IMAGE                        STATUS      NAMES
abc123        localhost/pgvector:latest    Running     pgvector-db
def456        localhost/llamaindex:latest  Running     llamaindex-service
```

---

## 7. Error Handling

### Common Errors and Exit Codes

| Exit Code | Error Category | Common Causes |
|-----------|----------------|---------------|
| 1 | Invalid Arguments | Missing required params, invalid file paths |
| 2 | Database Error | Connection failed, auth error, database doesn't exist |
| 3 | Service Unavailable | LLM service down, Ollama not running, API key invalid |
| 4 | Data Error | Empty results, no documents, no chunks found |
| 5 | Compatibility Error | Embedding dimension mismatch, model version conflict |

### Error Message Format

**stderr Output**:
```
ERROR: Database connection failed
  Cause: Connection refused to localhost:5432
  Solution: Ensure PostgreSQL is running and accessible
  Command: systemctl status postgresql
```

---

## 8. Configuration File Format

### config.yaml

Referenced by multiple CLI tools.

**Schema**:
```yaml
llm:
  provider: string  # "ollama" | "openai" | "anthropic"
  ollama: {...}
  openai: {...}
  anthropic: {...}

embedding:
  provider: string  # "huggingface" | "ollama" | "openai"
  huggingface: {...}

retrieval:
  top_k: integer (1-100)
  similarity_threshold: float (0.0-1.0)
  chunk_size: integer (128-2048)
  chunk_overlap: integer (0-chunk_size/2)

database:
  table_name: string
```

See `contracts/config-schema.yaml` for full specification.

---

## 9. CLI Composition Examples

### End-to-End Workflow

```bash
# Step 1: Ingest documents
python ingestion/ingest_documents.py \
  --input-dir ./examples/sample_docs \
  --output-db postgresql://postgres:password@localhost:5432/ragdb

# Step 2: Create dump
export POSTGRES_PASSWORD=password
./ingestion/create_dump.sh ragdb ragdb_dump.sql

# Step 3: Deploy containers (loads dump)
podman play kube pod.yaml

# Step 4: Wait for containers to be ready
sleep 30

# Step 5: Health check
python llamaindex-service/health_check.py

# Step 6: Execute query
python llamaindex-service/query_cli.py \
  --query "What is pgvector?" \
  --explain
```

### Batch Query Processing

```bash
# Process multiple queries from file
while IFS= read -r query; do
  python llamaindex-service/query_cli.py \
    --query "$query" \
    --json >> results.jsonl
done < queries.txt

# Aggregate results
cat results.jsonl | jq -r '.query + " -> " + .answer'
```

---

## 10. Backwards Compatibility

This is version 1.0.0 (initial release). Future versions will follow semantic versioning:
- **Major version**: Breaking CLI changes (argument removal, incompatible output format)
- **Minor version**: New features (new flags, additional output modes)
- **Patch version**: Bug fixes, documentation updates

---

This CLI interface provides a complete command-line experience for ingesting documents, deploying containers, and querying the RAG system.
