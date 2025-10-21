# Quickstart Guide: LlamaIndex + pgvectordb RAG Demo

**Feature**: LlamaIndex + pgvectordb RAG Demo
**Version**: 1.0.0
**Target Audience**: Developers and data scientists new to RAG systems
**Estimated Time**: 30 minutes (excluding model downloads)

This guide walks you through setting up and running the RAG demo from start to finish.

---

## Prerequisites

### Required Software

- **Python 3.11+**: `python --version`
- **PostgreSQL 15+** with **pgvector** extension: `psql --version`
- **Podman** or **Docker**: `podman --version` or `docker --version`
- **Git**: `git --version`

### Optional (for local LLM)

- **Ollama**: For running local LLMs ([install guide](https://ollama.ai))
  ```bash
  curl -fsSL https://ollama.ai/install.sh | sh
  ollama pull llama3.1:latest
  ```

### System Requirements

**Minimum**:
- 8GB RAM
- 10GB free disk space
- 4 CPU cores

**Recommended**:
- 16GB RAM
- 20GB free disk space
- 8 CPU cores
- GPU (optional, for faster embeddings)

---

## Quick Setup (5 minutes)

### Step 1: Clone Repository

```bash
git clone <repository-url>
cd demos/llamaindex-rag-demo
```

### Step 2: Install Python Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Step 3: Set Up Environment Variables

```bash
# Copy example environment file
cp .env.example .env

# Edit .env and set required variables
nano .env
```

**.env contents**:
```bash
# Database (required)
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_secure_password
POSTGRES_DB=ragdb

# API Keys (optional, only if using cloud LLMs)
# OPENAI_API_KEY=sk-...
# ANTHROPIC_API_KEY=sk-ant-...
```

### Step 4: Start PostgreSQL with pgvector

```bash
# Option 1: Using existing PostgreSQL
sudo systemctl start postgresql
psql -U postgres -c "CREATE EXTENSION IF NOT EXISTS vector;"

# Option 2: Using Docker/Podman
podman run -d \
  --name pgvector-db \
  -e POSTGRES_PASSWORD=your_secure_password \
  -e POSTGRES_DB=ragdb \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

### Step 5: Verify Installation

```bash
python ingestion/ingest_documents.py --help
python llamaindex-service/query_cli.py --help
```

---

## End-to-End Workflow (15 minutes)

### Step 1: Ingest Sample Documents

The demo includes sample documents about pgvector and vector databases.

```bash
# Ingest documents into database
python ingestion/ingest_documents.py \
  --input-dir examples/sample_docs \
  --output-db postgresql://postgres:${POSTGRES_PASSWORD}@localhost:5432/ragdb \
  --verbose
```

**Expected Output**:
```json
{
  "status": "success",
  "documents_processed": 8,
  "chunks_created": 247,
  "embeddings_generated": 247,
  "database_size_mb": 32.5,
  "processing_time_seconds": 18.3
}
```

**What this does**:
- Processes all documents in `examples/sample_docs/`
- Chunks documents into 512-token segments with 20% overlap
- Generates embeddings using all-MiniLM-L6-v2 (384 dimensions)
- Stores everything in PostgreSQL with pgvector extension

### Step 2: Create Database Dump (Optional)

Create a portable database dump for deployment.

```bash
export POSTGRES_PASSWORD=your_secure_password

./ingestion/create_dump.sh ragdb ragdb_dump.sql localhost 5432 postgres
```

**Output**:
```
Creating database dump...
Dumping schema...
Dumping data...
Dump created: ragdb_dump.sql (32.5 MB)
```

**What this does**:
- Creates a complete PostgreSQL dump including schema and data
- Dump file can be loaded into container for deployment
- Idempotent: can be re-run without errors

### Step 3: Configure LLM Provider

Edit `config.yaml` to select your LLM provider.

**Option A: Local (Ollama) - No API keys needed**:
```yaml
llm:
  provider: "ollama"
  ollama:
    model: "llama3.1:latest"
    request_timeout: 120.0
    base_url: "http://localhost:11434"

embedding:
  provider: "huggingface"
  huggingface:
    model_name: "sentence-transformers/all-MiniLM-L6-v2"
```

**Option B: Cloud (OpenAI)**:
```yaml
llm:
  provider: "openai"
  openai:
    model: "gpt-4o-mini"
    temperature: 0.7

embedding:
  provider: "huggingface"
  huggingface:
    model_name: "sentence-transformers/all-MiniLM-L6-v2"
```

Don't forget to set `OPENAI_API_KEY` in `.env`!

**Option C: Cloud (Anthropic)**:
```yaml
llm:
  provider: "anthropic"
  anthropic:
    model: "claude-sonnet-4-0"
    max_tokens: 1024

embedding:
  provider: "huggingface"
  huggingface:
    model_name: "sentence-transformers/all-MiniLM-L6-v2"
```

Don't forget to set `ANTHROPIC_API_KEY` in `.env`!

### Step 4: Run Health Check

Verify everything is configured correctly.

```bash
python llamaindex-service/health_check.py --verbose
```

**Expected Output**:
```
RAG System Health Check
=======================

[✓] Configuration file: config.yaml
[✓] Database connection: ragdb@localhost:5432
[✓] Database view: data_llamaindex (247 rows)
[✓] Embedding model: sentence-transformers/all-MiniLM-L6-v2 (384 dims)
[✓] LLM service: ollama (llama3.1:latest)
[✓] Vector dimensions: 384 (matches embedding model)

All checks passed!
```

### Step 5: Execute Your First Query

```bash
python llamaindex-service/query_cli.py \
  --query "What is pgvector and how does it work?"
```

**Expected Output**:
```
Query: What is pgvector and how does it work?

Answer:
pgvector is a PostgreSQL extension that enables vector similarity search
directly within PostgreSQL databases. It provides efficient storage and
retrieval of high-dimensional vectors through specialized data types and
indexes. The extension works by storing embeddings as vector types and
using optimized similarity algorithms like cosine distance to find
semantically similar content.

Sources:
[1] pgvector_intro.pdf (page 1, similarity: 0.89)
[2] vector_databases.md (chunk 2, similarity: 0.84)
[3] postgres_extensions.pdf (page 3, similarity: 0.78)

Query time: 2.1s
```

**Congratulations!** You've just executed your first RAG query!

---

## Exploring Features (10 minutes)

### Interactive Chat Mode

Start a conversational session with your documents.

```bash
python llamaindex-service/query_cli.py --interactive
```

**Example session**:
```
RAG Interactive Chat
Type /help for commands, /exit to quit

> What is pgvector?
[Answer about pgvector...]

> How do I install it?
[Answer about installation...]

> What are the performance characteristics?
[Answer about performance...]

> /sources
[Shows sources from last query]

> /explain
[Shows detailed retrieval information]

> /exit
Goodbye!
```

### Explain Mode (Deep Dive)

See exactly how the RAG system works under the hood.

```bash
python llamaindex-service/query_cli.py \
  --query "What is pgvector?" \
  --explain
```

**Output**:
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
  [1] Similarity: 0.89 | pgvector_intro.pdf (chunk 0)
      Text: "pgvector is a PostgreSQL extension that..."
  [2] Similarity: 0.84 | vector_databases.md (chunk 2)
      Text: "Vector databases like pgvector enable..."

Step 3: LLM Generation
  Provider: ollama
  Model: llama3.1:latest
  Context length: 1842 tokens
  Temperature: 0.7
  Generation time: 1.8s

Answer:
[Generated answer...]

Total time: 2.1s
```

### JSON Output Mode

Get machine-readable output for integration.

```bash
python llamaindex-service/query_cli.py \
  --query "What is pgvector?" \
  --json
```

**Output**:
```json
{
  "query": "What is pgvector?",
  "answer": "pgvector is a PostgreSQL extension...",
  "sources": [
    {
      "document": "pgvector_intro.pdf",
      "chunk_index": 0,
      "similarity_score": 0.89,
      "metadata": {"page_number": 1}
    }
  ],
  "llm_provider": "ollama",
  "llm_model": "llama3.1:latest",
  "query_time_seconds": 2.1,
  "retrieval_count": 5,
  "embedding_model": "sentence-transformers/all-MiniLM-L6-v2"
}
```

### Custom Retrieval Parameters

Fine-tune retrieval for your needs.

```bash
# Get more results with lower threshold
python llamaindex-service/query_cli.py \
  --query "What is pgvector?" \
  --top-k 10 \
  --similarity-threshold 0.3

# Get fewer, more relevant results
python llamaindex-service/query_cli.py \
  --query "What is pgvector?" \
  --top-k 3 \
  --similarity-threshold 0.8
```

---

## Using Your Own Documents

### Step 1: Prepare Your Documents

Supported formats:
- PDF (`.pdf`)
- Markdown (`.md`)
- Text (`.txt`)
- HTML (`.html`)

**Organize documents**:
```bash
mkdir -p my_documents
cp /path/to/your/docs/* my_documents/
```

### Step 2: Ingest Your Documents

```bash
python ingestion/ingest_documents.py \
  --input-dir my_documents \
  --output-db postgresql://postgres:${POSTGRES_PASSWORD}@localhost:5432/ragdb \
  --verbose
```

### Step 3: Create Dump (Optional)

```bash
./ingestion/create_dump.sh ragdb my_ragdb_dump.sql
```

### Step 4: Query Your Documents

```bash
python llamaindex-service/query_cli.py \
  --query "Your question about your documents"
```

---

## Container Deployment (Production)

### Step 1: Build Container Images

```bash
# Build pgvector container with dump loading
cd pgvector
podman build -t ragdb-pgvector:latest .

# Build LlamaIndex service container
cd ../llamaindex-service
podman build -t ragdb-llamaindex:latest .
```

### Step 2: Update pod.yaml

Edit `pod.yaml` to reference your dump file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rag-demo-pod
spec:
  containers:
  - name: pgvector-db
    image: localhost/ragdb-pgvector:latest
    env:
    - name: POSTGRES_PASSWORD
      value: "your_secure_password"
    - name: DUMP_FILE
      value: "/dumps/ragdb_dump.sql"
    volumeMounts:
    - name: db-dumps
      mountPath: /dumps
    ports:
    - containerPort: 5432

  - name: llamaindex-service
    image: localhost/ragdb-llamaindex:latest
    env:
    - name: POSTGRES_HOST
      value: "localhost"
    - name: POSTGRES_PASSWORD
      value: "your_secure_password"
    ports:
    - containerPort: 8000

  volumes:
  - name: db-dumps
    hostPath:
      path: /path/to/dumps
      type: Directory
```

### Step 3: Deploy Pod

```bash
podman play kube pod.yaml
```

### Step 4: Verify Deployment

```bash
# Check pod status
podman pod ps
podman ps --pod

# Check logs
podman logs <pgvector-container-id>
podman logs <llamaindex-container-id>

# Test query
podman exec -it <llamaindex-container-id> \
  python query_cli.py --query "What is pgvector?"
```

---

## Troubleshooting

### Database Connection Failed

**Error**: `Connection refused to localhost:5432`

**Solutions**:
1. Verify PostgreSQL is running: `systemctl status postgresql`
2. Check connection parameters in `.env`
3. Test connection: `psql -h localhost -U postgres -d ragdb`

### Embedding Model Not Found

**Error**: `Model 'all-MiniLM-L6-v2' not found`

**Solution**:
```bash
# Model downloads automatically on first use
# Ensure internet connection and sufficient disk space
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')"
```

### Ollama Service Unavailable

**Error**: `Connection refused to http://localhost:11434`

**Solutions**:
1. Start Ollama: `ollama serve`
2. Pull model: `ollama pull llama3.1:latest`
3. Verify: `curl http://localhost:11434`

### Vector Dimension Mismatch

**Error**: `Dimension mismatch: expected 384, got 1536`

**Solution**:
- Ensure embedding model matches database
- Check `config.yaml` embedding provider
- Verify docs2db used same embedding model
- If changed, re-ingest documents

### No Results Found

**Error**: `No relevant results found for query`

**Solutions**:
1. Lower similarity threshold: `--similarity-threshold 0.3`
2. Increase top-k: `--top-k 10`
3. Verify documents were ingested: `psql -d ragdb -c "SELECT COUNT(*) FROM chunks;"`
4. Check query embedding: Use `--explain` flag

### Out of Memory

**Error**: `MemoryError` or container OOM killed

**Solutions**:
1. Reduce context_window in `config.yaml`
2. Use smaller model (e.g., llama2:7b instead of 13b)
3. Reduce top_k retrieval parameter
4. Increase system RAM or swap

---

## Performance Tuning

### Query Speed

**Faster queries**:
- Use smaller embedding model (but maintain consistency!)
- Reduce `top_k` (fewer chunks to retrieve)
- Increase `similarity_threshold` (fewer low-quality results)
- Use Ollama local LLM (no API latency)

**Better quality**:
- Use larger, more capable LLM (GPT-4, Claude Opus)
- Increase `top_k` (more context for LLM)
- Decrease `similarity_threshold` (more potential sources)

### Ingestion Speed

**Faster ingestion**:
- Use GPU for embedding generation
- Increase batch size in embedding config
- Use faster embedding model (e.g., all-MiniLM vs all-mpnet)

### Memory Usage

**Reduce memory**:
- Use quantized models (e.g., llama3.1:latest:q4)
- Limit context_window
- Process fewer documents at once

---

## Next Steps

### Learn More

- **Architecture Deep Dive**: See `architecture.md` for system design details
- **API Reference**: See `contracts/python-api.md` for programmatic access
- **Advanced Configuration**: See `contracts/config-schema.yaml` for all options

### Customize

1. **Add Custom Documents**: Ingest your own document corpus
2. **Try Different LLMs**: Compare Ollama, OpenAI, and Anthropic models
3. **Tune Retrieval**: Experiment with top-k and similarity thresholds
4. **Build Applications**: Use Python API to integrate RAG into your apps

### Compare Approaches

This demo uses LlamaIndex. Compare with:
- **Llama Stack Demo**: Similar RAG demo using Llama Stack framework
- **Direct pgvector**: Query pgvector directly without LlamaIndex
- See `architecture.md` for decision criteria

---

## Getting Help

### Documentation

- README.md: Overview and features
- architecture.md: System design and data flow
- contracts/: API specifications

### Issues

- Check existing issues on GitHub
- File bug reports with reproducible examples
- Request features with use case descriptions

### Community

- Join discussions in GitHub Discussions
- Share your implementations and use cases

---

## Summary

You've learned how to:
- ✅ Set up the RAG demo environment
- ✅ Ingest documents and create vector embeddings
- ✅ Configure LLM providers (local and cloud)
- ✅ Execute queries and interpret results
- ✅ Deploy containers for production use
- ✅ Troubleshoot common issues

**Total time**: ~30 minutes (excluding model downloads)

**Next**: Try ingesting your own documents and building a custom RAG application!
