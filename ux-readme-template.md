# LlamaIndex + pgvectordb RAG Demo

**UX-Approved README Template**
**Note**: This is a TEMPLATE showing structure and style. Replace bracketed content with actual implementation.

---

# [Demo Name]: LlamaIndex RAG with PostgreSQL pgvector

[One-sentence description: What this demo does and why it matters]

Example: "End-to-end demonstration of Retrieval-Augmented Generation using LlamaIndex with PostgreSQL pgvector for document search and LLM-powered question answering."

---

## Quick Start

**Time to first success**: ~10 minutes

### 1. Verify Prerequisites

Check that you have the required services:

```bash
# Check Python version (need 3.12+)
python --version

# Check Ollama is running
curl http://localhost:11434

# Check PostgreSQL with pgvector
psql -c "SELECT * FROM pg_extension WHERE extname = 'vector';"
```

**Expected output**: [Show what success looks like]

### 2. Setup Demo Environment

Create an isolated environment with all dependencies:

```bash
# From: /path/to/docs2db-api/demos/llamaindex-rag
uv run python setup_llamaindex_rag.py my-demo
```

**What this does**: Creates `my-demo/` directory with configured LlamaIndex, database connection, and sample documents.

**Expected output**:
```
[SETUP] Creating demo environment...
[SETUP] Installing dependencies...
[SETUP] Configuring database...
[SETUP] Loading sample documents...
[SUCCESS] Demo ready! Next: cd my-demo && ./start.sh
```

### 3. Start Services

```bash
# From: /path/to/my-demo
./start.sh
```

**Expected output**: "Server running on http://localhost:8000"

### 4. Run Demo Query

```bash
# From: /path/to/my-demo
uv run python client.py
```

**Expected output**: [Show example query and response]

**Congratulations!** You've just run a complete RAG pipeline. The system:
1. Ingested documents into PostgreSQL with vector embeddings
2. Processed your query using LlamaIndex
3. Retrieved relevant document chunks via vector similarity
4. Generated a response using the LLM with document context

---

## What You'll Build

This demo creates a complete RAG system with:

- **Document ingestion pipeline**: Upload and process various document types
- **Vector database**: PostgreSQL with pgvector for semantic search
- **LLM integration**: Query processing with context-aware responses
- **API interface**: RESTful endpoints for querying your documents

**Value**: Learn how to build production-ready document Q&A systems with open-source tools.

---

## Prerequisites

### Required Services

| Service | Version | Purpose | Verification |
|---------|---------|---------|--------------|
| Python | 3.12+ | Runtime | `python --version` |
| PostgreSQL | 14+ | Vector database | `psql --version` |
| pgvector | 0.5+ | Vector similarity | `psql -c "SELECT * FROM pg_extension WHERE extname = 'vector';"` |
| Ollama | Latest | LLM inference | `curl http://localhost:11434` |
| uv | Latest | Package management | `uv --version` |

### Required Models

```bash
# Pull the model used in demo
ollama pull llama3.2:3b-instruct-fp16
```

### System Requirements

- **Minimum**: 4GB RAM, 10GB disk space
- **Recommended**: 8GB RAM, 20GB disk space
- **Why**: LLM models require ~2-3GB RAM, vector embeddings require ~5GB disk for moderate document corpus

### Installation Guides

If you're missing prerequisites:
- [Install Python 3.12+](https://python.org)
- [Install PostgreSQL with pgvector](https://github.com/pgvector/pgvector#installation)
- [Install Ollama](https://ollama.ai/download)
- [Install uv](https://docs.astral.sh/uv/getting-started/installation/)

---

## Usage Examples

### Example 1: Basic Query

```bash
# Query the pre-loaded sample documents
uv run python client.py --query "What is retrieval-augmented generation?"
```

### Example 2: Upload Your Own Documents

```bash
# Ingest your documents
uv run python ingest.py --path /path/to/your/docs

# Query your documents
uv run python client.py --query "Your question about your docs"
```

### Example 3: Customize the Model

```bash
# Use a different Ollama model
uv run python client.py --model qwen2.5:7b-instruct --query "Your question"
```

### Example 4: Agent Integration

```bash
# Run the agent demo that uses RAG as a tool
uv run python agent_example.py
```

**What this shows**: How to integrate the RAG system as a tool in an autonomous agent workflow.

---

## Architecture

### Component Overview

```
[User Query]
     |
     v
[LlamaIndex Query Engine]
     |
     +---> [Vector Embeddings] ---> [PostgreSQL pgvector]
     |                                      |
     |                              [Retrieve Similar Docs]
     |                                      |
     +---> [LLM (via Ollama)] <--- [Context + Query]
     |
     v
[Generated Response]
```

### Data Flow

1. **Document Ingestion** (one-time setup):
   - Parse documents (PDF, TXT, Markdown, etc.)
   - Chunk text into semantic segments
   - Generate embeddings using LlamaIndex
   - Store vectors in PostgreSQL with pgvector

2. **Query Processing** (per request):
   - Embed user query using same model
   - Perform vector similarity search in database
   - Retrieve top-k relevant document chunks
   - Construct prompt with retrieved context
   - Generate response using LLM

3. **Response Generation**:
   - LLM receives query + relevant document context
   - Generates grounded response based on documents
   - Returns response with citations (optional)

### Technology Stack

- **LlamaIndex**: Orchestration framework for RAG workflows
- **PostgreSQL + pgvector**: Vector database for semantic search
- **Ollama**: Local LLM inference engine
- **Python 3.12+**: Runtime environment
- **uv**: Fast Python package and project manager

---

## Troubleshooting

### Issue: "Connection refused on port 11434"

**Cause**: Ollama is not running

**Solution**:
```bash
# Start Ollama
ollama serve

# Verify it's running
curl http://localhost:11434
```

### Issue: "psycopg2.OperationalError: could not connect to server"

**Cause**: PostgreSQL is not running or connection details are incorrect

**Solution**:
```bash
# Check PostgreSQL status
systemctl status postgresql

# If not running, start it
systemctl start postgresql

# Verify connection
psql -U postgres -c "SELECT version();"
```

### Issue: "pgvector extension not found"

**Cause**: pgvector extension not installed in PostgreSQL

**Solution**:
```bash
# Install pgvector (Ubuntu/Debian)
sudo apt install postgresql-16-pgvector

# Install pgvector (RHEL/Fedora)
sudo dnf install pgvector_16

# Enable in database
psql -U postgres -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### Issue: "Model 'llama3.2:3b-instruct-fp16' not found"

**Cause**: Required Ollama model not pulled

**Solution**:
```bash
# Pull the model
ollama pull llama3.2:3b-instruct-fp16

# Verify it's available
ollama list
```

### Issue: "Setup script fails with 'demo-dir already exists'"

**Cause**: Demo directory from previous run exists

**Solution**:
```bash
# Option 1: Use a different directory name
uv run python setup_llamaindex_rag.py my-demo-2

# Option 2: Clean up and retry
rm -rf my-demo
uv run python setup_llamaindex_rag.py my-demo
```

### Issue: "Out of memory during query"

**Cause**: System doesn't have enough RAM for the model

**Solution**:
```bash
# Option 1: Use a smaller model
uv run python client.py --model llama3.2:1b

# Option 2: Reduce context window size
uv run python client.py --context-size 2048

# Option 3: Add swap space (temporary)
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Still Having Issues?

1. **Check logs**: `./logs.sh` or `journalctl -u postgresql`
2. **Verify environment**: `./check-health.sh`
3. **Review configuration**: Check `config.yaml` for correct settings
4. **Ask for help**: Open an issue with:
   - Error message (full text)
   - Output of `./check-health.sh`
   - Operating system and version

---

## Advanced Topics

<details>
<summary>Custom Document Processing</summary>

### Adding Custom Document Loaders

LlamaIndex supports many document types. To add custom processing:

```python
from llama_index import SimpleDirectoryReader

# Configure custom file extensions
documents = SimpleDirectoryReader(
    input_dir="./docs",
    file_extractor={
        ".custom": CustomParser(),
    }
).load_data()
```

See [LlamaIndex documentation](https://docs.llamaindex.ai) for full details.

</details>

<details>
<summary>Performance Tuning</summary>

### Optimization Strategies

1. **Chunk size**: Adjust for your document type
   - Technical docs: 512-1024 tokens
   - Narrative text: 1024-2048 tokens

2. **Top-k retrieval**: Balance relevance vs. context size
   - Start with k=5, adjust based on results

3. **Embedding model**: Use larger models for better accuracy
   - Default: `text-embedding-3-small` (fast)
   - Better: `text-embedding-3-large` (accurate)

4. **Database indexing**: Ensure vector indexes are created
   ```sql
   CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops);
   ```

</details>

<details>
<summary>Production Deployment</summary>

### Production Considerations

- **Security**: Use environment variables for credentials, not hardcoded values
- **Monitoring**: Add logging and metrics (Prometheus, Grafana)
- **Scalability**: Consider managed vector databases (Pinecone, Weaviate) for scale
- **Rate limiting**: Protect LLM endpoints from abuse
- **Caching**: Cache embeddings and common queries
- **Backup**: Regular backups of vector database

See `PRODUCTION.md` for detailed guidance.

</details>

---

## Related Resources

- **LlamaIndex Documentation**: https://docs.llamaindex.ai
- **pgvector Documentation**: https://github.com/pgvector/pgvector
- **Ollama Models**: https://ollama.ai/library
- **RAG Best Practices**: [Link to internal docs]

---

## Sample Data Attribution

Sample documents included in this demo:
- [Document 1]: [Source and license]
- [Document 2]: [Source and license]

All sample data is used under permissive licenses for educational purposes.

---

## Contributing

Found an issue or want to improve this demo?

1. Open an issue describing the problem or enhancement
2. Submit a pull request with your changes
3. Ensure all tests pass and documentation is updated

See `CONTRIBUTING.md` for detailed guidelines.

---

## License

[License information]

---

**Questions?** Open an issue or contact the docs2db team.

**Feedback?** Let us know how we can improve this demo!
