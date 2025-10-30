# Quick Start: Llamaindex + RamaLama RAG Demo

**Goal**: Deploy a working RAG system in under 30 minutes

**What You'll Build**: A containerized RAG demonstration that queries your document collection using Llamaindex orchestration and local LLM serving via RamaLama.

---

## Prerequisites

Before starting, ensure you have:

### Required
- **Container Runtime**: Podman 4.0+ or Docker 20.10+
- **System Resources**: 4GB RAM minimum, 20GB free disk space
- **Operating System**: Linux (native) or macOS/Windows (with Podman/Docker Desktop)
- **Network**: Internet connection for initial model downloads

### Recommended
- **CPU**: 4+ cores for acceptable inference speed
- **Storage**: SSD for faster model loading
- **Terminal**: Bash or compatible shell

### Verify Prerequisites

```bash
# Check container runtime
podman --version  # or: docker --version

# Check available resources
free -h  # Linux
# macOS: Activity Monitor → Memory tab
# Windows: Task Manager → Performance → Memory

# Check disk space
df -h
```

---

## Step 1: Prepare Document Database (5 minutes)

The demo requires a pre-processed database created by the docs2db pipeline.

### Option A: Use Sample Database (Fastest)

```bash
# Download sample database dump
curl -LO https://example.com/sample-ragdb.sql.gz
gunzip sample-ragdb.sql.gz

# Verify download
ls -lh sample-ragdb.sql  # Should be ~10-50 MB
```

**Sample Database Contains**:
- 30 technical documents (mixed PDF, HTML, Markdown)
- ~1,000 chunks
- Embeddings from `granite-30m-english` model (384 dimensions)

### Option B: Process Your Own Documents

If you have your own document collection:

```bash
# Clone docs2db-api repository
git clone https://github.com/your-org/docs2db-api.git
cd docs2db-api

# Process documents (see docs2db-api README for details)
docs2db embed --model granite-30m-english --input /path/to/documents
docs2db load --output ragdb_dump.sql

# Copy dump to demo directory
cp ragdb_dump.sql /path/to/rag-demo/
```

**Note**: Document processing time varies (2-30 minutes depending on collection size).

---

## Step 2: Deploy PostgreSQL + pgvector (3 minutes)

The vector database stores document embeddings and provides similarity search.

### Start Database Container

```bash
# Create demo directory
mkdir -p llamaindex-rag-demo
cd llamaindex-rag-demo

# Start PostgreSQL with pgvector
podman run -d \
  --name rag-postgres \
  -e POSTGRES_PASSWORD=demo123 \
  -e POSTGRES_DB=ragdb \
  -p 5432:5432 \
  -v $(pwd)/sample-ragdb.sql:/docker-entrypoint-initdb.d/init.sql:ro \
  pgvector/pgvector:pg14

# Wait for database to initialize (30-60 seconds)
sleep 60

# Verify database is ready
podman exec rag-postgres pg_isready
```

### Load Database (if not using init script)

If your database dump wasn't loaded automatically:

```bash
# Copy dump into container
podman cp sample-ragdb.sql rag-postgres:/tmp/

# Load dump
podman exec -i rag-postgres psql -U postgres -d ragdb < sample-ragdb.sql
```

### Verify Database

```bash
# Check document count
podman exec rag-postgres psql -U postgres -d ragdb -c \
  "SELECT COUNT(*) FROM documents;"

# Check embedding model
podman exec rag-postgres psql -U postgres -d ragdb -c \
  "SELECT name, dimensions FROM models;"
```

**Expected Output**:
```
 name                                      | dimensions
-------------------------------------------+------------
 ibm-granite/granite-embedding-30m-english | 384
```

---

## Step 3: Deploy RamaLama LLM Server (5-10 minutes)

RamaLama serves the language model for response generation.

### Pre-pull Model (Recommended)

Pre-pulling the model avoids waiting during first query:

```bash
# Pull TinyLlama model (~1.7 GB, 2-5 minutes depending on connection)
ramalama pull ollama://tinyllama:1.1b-chat-q4_k_m

# Verify model
ramalama list | grep tinyllama
```

### Start RamaLama Server

```bash
# Start model server
ramalama serve \
  --name rag-llm \
  --port 8080 \
  --ngl 0 \
  --threads 4 \
  --ctx-size 2048 \
  ollama://tinyllama:1.1b-chat-q4_k_m

# Server will start in foreground. Open new terminal for next steps.
# Or run in background: add & at end and use `podman logs -f rag-llm` to monitor
```

**Startup Time**: 40-80 seconds (model loading)

### Verify LLM Server

In a new terminal:

```bash
# Check health endpoint
curl http://localhost:8080/v1/models

# Expected output: JSON with model information
```

---

## Step 4: Deploy Llamaindex RAG Service (5 minutes)

The RAG service orchestrates retrieval and generation.

### Create Configuration

```bash
# Create config file
cat > config.yaml <<EOF
database:
  host: localhost
  port: 5432
  database: ragdb
  user: postgres
  password: demo123

embedding:
  model: granite-30m-english
  dimensions: 384

llm:
  endpoint: http://localhost:8080/v1
  model: tinyllama
  timeout: 30

retrieval:
  top_k: 5
  similarity_threshold: 0.7
  enable_hybrid: false

server:
  host: 0.0.0.0
  port: 8000
EOF
```

### Build and Start RAG Service

```bash
# Clone demo repository (or use docs2db-api/demos/llamaindex-ramalama)
git clone https://github.com/your-org/llamaindex-rag-demo.git
cd llamaindex-rag-demo

# Build container image
podman build -t rag-demo:latest .

# Start service
podman run -d \
  --name rag-service \
  --network host \
  -v $(pwd)/config.yaml:/app/config.yaml:ro \
  rag-demo:latest

# Monitor startup logs
podman logs -f rag-service
```

**Look for**:
```
✓ Embedding model validated: granite-30m-english (384D)
✓ Database connected: 1247 embeddings loaded
✓ LLM server ready: tinyllama-1.1b
✓ Llamaindex server ready
INFO:     Uvicorn running on http://0.0.0.0:8000
```

**Startup Time**: ~60-90 seconds

---

## Step 5: Test the System (2 minutes)

### Health Check

```bash
# Check system health
curl http://localhost:8000/health | jq

# Expected output
{
  "status": "healthy",
  "components": {
    "database": {
      "status": "connected",
      "latency_ms": 12
    },
    "embedding_model": {
      "status": "validated",
      "model": "granite-30m-english",
      "dimensions": 384
    },
    "llm_server": {
      "status": "loaded",
      "model": "tinyllama-1.1b"
    }
  }
}
```

### Database Statistics

```bash
# View system stats
curl http://localhost:8000/stats | jq

# Sample output
{
  "database": {
    "documents": 30,
    "chunks": 1247,
    "embeddings": 1247,
    "embedding_model": {
      "name": "granite-30m-english",
      "dimensions": 384
    }
  }
}
```

### Submit Test Query

```bash
# Basic query
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the system requirements?"
  }' | jq

# Expected response (2-5 seconds)
{
  "query_id": "a3bb189e-8bf9-4b5a-b6b5-8a21a86f7e4f",
  "answer": "The system requires Python 3.12 or higher, 4GB RAM, and 20GB storage. [Source: installation_guide.pdf, chunk 3]",
  "sources": [
    {
      "chunk_id": 145,
      "similarity_score": 0.87,
      "source_file": "installation_guide.pdf"
    }
  ],
  "confidence": "high",
  "processing_time_ms": 3247
}
```

---

## Step 6: Try Example Queries (5 minutes)

### Query with Custom Parameters

```bash
# Higher precision (stricter threshold)
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How do I configure authentication?",
    "top_k": 7,
    "similarity_threshold": 0.75
  }' | jq
```

### Query About Non-Indexed Content

```bash
# Test out-of-scope handling
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is the weather in Tokyo?"
  }' | jq

# Expected: "I don't have information about that..."
```

### Streaming Response

```bash
# Stream response with SSE
curl -N -X POST http://localhost:8000/query/stream \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Explain the architecture"
  }'

# Output: Progressive token stream
event: sources
data: {"sources": [...]}

event: token
data: {"delta": "The", "accumulated": "The"}

event: token
data: {"delta": " architecture", "accumulated": "The architecture"}
...
```

---

## Troubleshooting

### Issue: Database Connection Failed

**Symptom**: Health check shows `"database": {"status": "disconnected"}`

**Solution**:
```bash
# Check PostgreSQL is running
podman ps | grep rag-postgres

# Check logs
podman logs rag-postgres

# Verify port binding
podman port rag-postgres

# Test connection directly
podman exec rag-postgres pg_isready
```

### Issue: Embedding Model Mismatch

**Symptom**: Startup error "Embedding dimension mismatch"

**Solution**:
```bash
# Check database model
podman exec rag-postgres psql -U postgres -d ragdb -c \
  "SELECT name, dimensions FROM models;"

# Update config.yaml to match:
embedding:
  model: [model_name_from_database]
  dimensions: [dimensions_from_database]

# Restart service
podman restart rag-service
```

### Issue: LLM Server Unavailable

**Symptom**: Health check shows `"llm_server": {"status": "unavailable"}`

**Solution**:
```bash
# Check RamaLama is running
podman ps | grep rag-llm
# Or check process: ps aux | grep ramalama

# Check endpoint
curl http://localhost:8080/v1/models

# Restart RamaLama if needed
ramalama serve --port 8080 ollama://tinyllama:1.1b-chat-q4_k_m
```

### Issue: Slow Query Responses (>10 seconds)

**Possible Causes**:

1. **No HNSW Index**: Create vector index for faster search
   ```bash
   podman exec rag-postgres psql -U postgres -d ragdb -c \
     "CREATE INDEX IF NOT EXISTS embeddings_hnsw_idx ON embeddings USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);"
   ```

2. **CPU Overload**: Reduce concurrent load or increase CPU cores

3. **Large Context**: Reduce `top_k` parameter
   ```bash
   curl -X POST http://localhost:8000/query \
     -d '{"query": "...", "top_k": 3}'  # Instead of 5
   ```

### Issue: "Connection refused" Errors

**Symptom**: Services can't connect to each other

**Solution**:
```bash
# Verify all containers use --network host
# Or create shared network:
podman network create rag-network

# Restart containers with:
podman run --network rag-network ...

# Update config.yaml endpoints:
database:
  host: rag-postgres  # Instead of localhost
llm:
  endpoint: http://rag-llm:8080/v1  # Instead of localhost
```

---

## Next Steps

### Explore the System

1. **API Documentation**: Open http://localhost:8000/docs (Swagger UI)
2. **Monitoring**: Watch logs with `podman logs -f rag-service`
3. **Tuning**: Adjust `config.yaml` parameters and restart

### Performance Tuning

```yaml
# config.yaml adjustments

retrieval:
  top_k: 7              # More context (slower)
  similarity_threshold: 0.65  # More results (lower precision)
  enable_hybrid: true   # Better recall (slower)

llm:
  timeout: 60           # Longer for complex queries
```

### Add Your Documents

1. Process your documents with docs2db-api
2. Export database dump: `pg_dump ragdb > my-docs.sql`
3. Restart PostgreSQL with your dump
4. Verify with `/stats` endpoint

### Production Deployment

See `docs/architecture.md` and `docs/production-deployment.md` for:
- OpenShift deployment configuration
- Horizontal scaling patterns
- Security hardening
- Performance optimization
- Monitoring and alerting

---

## Cleanup

Stop and remove all containers:

```bash
# Stop services
podman stop rag-service rag-postgres
pkill ramalama  # Or: podman stop rag-llm if containerized

# Remove containers
podman rm rag-service rag-postgres

# Optional: Remove images and data
podman rmi rag-demo:latest pgvector/pgvector:pg14
rm -rf llamaindex-rag-demo/
```

---

## Summary

You've successfully deployed:

- ✅ PostgreSQL + pgvector database with indexed documents
- ✅ RamaLama local LLM serving (TinyLlama 1.1B)
- ✅ Llamaindex RAG orchestration service
- ✅ REST API for querying document collection

**Query Endpoint**: http://localhost:8000/query
**API Docs**: http://localhost:8000/docs
**Health Check**: http://localhost:8000/health

**Total Setup Time**: 20-30 minutes (including model downloads)

---

## Additional Resources

- **API Reference**: See `contracts/openapi.yaml` for full API specification
- **Architecture**: See `../plan.md` for design details
- **Data Model**: See `../data-model.md` for entity definitions
- **Research**: See `../research.md` for technology decisions
- **docs2db Documentation**: https://github.com/your-org/docs2db-api

**Questions or Issues?** Create an issue in the project repository.
