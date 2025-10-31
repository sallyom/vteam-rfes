# Quickstart Guide: Llamaindex RAG Demo with Ramalama

**Feature**: 001-llamaindex-ramalama-demo
**Last Updated**: 2025-10-31

## Overview

This guide provides step-by-step instructions to deploy and test the Llamaindex RAG demo. The demo consists of 4 containerized services running in a Podman pod:

- **PostgreSQL + PGVector**: Vector database with pre-loaded docs2db embeddings
- **Ramalama**: Model serving for inference and embeddings
- **RAG Service**: Llamaindex orchestration layer (FastAPI)
- **Web UI**: Interactive Streamlit interface

**Expected deployment time**: 15-30 minutes (excluding model downloads)

---

## Prerequisites

### System Requirements
- **OS**: Linux, macOS, or Windows with WSL2
- **RAM**: 8GB minimum (16GB recommended)
- **Disk**: 20GB free space
- **Container Runtime**: Podman or Docker installed

### Software Dependencies
```bash
# Check Podman version
podman --version
# Minimum: 4.0+

# Check available resources
free -h  # Should show 8GB+ available RAM
df -h    # Should show 20GB+ free disk space
```

### Required Files
- `ragdb_dump.sql`: Pre-generated docs2db database dump
- Located at: `./data/ragdb_dump.sql` (or specify path during build)

---

## Quick Start (TL;DR)

```bash
# Clone repository
git clone <repo-url>
cd llamaindex-ramalama-demo

# Build and deploy (all-in-one)
make deploy

# Wait for services to be ready (~5 minutes)
make status

# Access web UI
open http://localhost:8501

# Test with example query
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What is renewable energy?"}'
```

---

## Detailed Deployment Steps

### Step 1: Build Container Images

Build all 4 container images from source:

```bash
# Navigate to project root
cd llamaindex-ramalama-demo

# Build all images
./scripts/build-containers.sh

# Or build individually:
podman build -t localhost/rag-pgvector:latest containers/database/
podman build -t localhost/ramalama-model:latest containers/ramalama/
podman build -t localhost/llamaindex-rag:latest containers/rag-service/
podman build -t localhost/rag-ui:latest containers/web-ui/
```

**Expected time**: 5-10 minutes

**Output verification**:
```bash
podman images | grep rag
# Should show 4 images with 'localhost' registry
```

---

### Step 2: Create Persistent Volumes

Create volumes for database and model cache:

```bash
# Create PVC for database
cat <<EOF | podman kube play -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pgvector-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
EOF

# Create PVC for model cache
cat <<EOF | podman kube play -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ramalama-models-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi
EOF
```

**Verify**:
```bash
podman volume ls
# Should show: pgvector-pvc and ramalama-models-pvc
```

---

### Step 3: Deploy Pod

Deploy all services using Podman Kubernetes YAML:

```bash
# Deploy ConfigMap (configuration)
podman kube play ./pod/configmap.yaml

# Deploy Pod (all services)
podman kube play ./pod/pod.yaml
```

**For Docker Compose (alternative)**:
```bash
docker-compose up -d
```

---

### Step 4: Monitor Startup

Watch services as they start:

```bash
# Check pod status
podman pod ps

# Check container status
podman ps --filter "pod=llamaindex-rag-demo"

# Watch logs
podman logs -f llamaindex-rag-demo-pgvector      # Database initialization
podman logs -f llamaindex-rag-demo-ramalama      # Model downloading/loading
podman logs -f llamaindex-rag-demo-rag-service   # RAG service startup
podman logs -f llamaindex-rag-demo-web-ui        # UI server
```

**What to expect**:
1. **Database** (~30 seconds): PostgreSQL initialization, dump restoration
2. **Ramalama** (~5-15 minutes, first run): Model download, loading into memory
3. **RAG Service** (~10 seconds): Connecting to DB, model server
4. **Web UI** (~5 seconds): Starting Streamlit server

---

### Step 5: Verify Health

Check that all services are healthy:

```bash
# Database
psql -h localhost -U postgres -d ragdb -c "SELECT COUNT(*) FROM data_documents;"
# Should return count > 0

# Model server
curl http://localhost:8080/health
# Should return {"status": "healthy"}

# RAG service
curl http://localhost:8000/health
# Should return {"status": "healthy"}

curl http://localhost:8000/ready
# Should return {"ready": true, "checks": {...}}

# Web UI
curl -I http://localhost:8501
# Should return HTTP 200
```

**Automated health check**:
```bash
./scripts/health-check.sh
# Runs all checks and reports status
```

---

### Step 6: Access Web Interface

Open the Streamlit UI in your browser:

```bash
# Linux/macOS
open http://localhost:8501

# Or navigate manually
# http://localhost:8501
```

**Interface features**:
- Text input for natural language queries
- Real-time query results with answers
- Source attribution with similarity scores
- Expandable source document sections

---

## Testing the System

### Example Queries

Try these sample queries in the web UI or via API:

**Basic query**:
```
What are the benefits of renewable energy?
```

**Specific query**:
```
How does wind energy compare to solar power?
```

**Technical query**:
```
What are the technical requirements for installing solar panels?
```

### API Testing

Test the REST API directly:

**Submit query**:
```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is renewable energy?",
    "similarity_threshold": 0.7,
    "max_chunks": 5,
    "query_mode": "hybrid"
  }'
```

**Expected response**:
```json
{
  "query_id": "query_abc123",
  "answer": "Renewable energy is energy from sources that are naturally replenishing...",
  "sources": [
    {
      "chunk_id": 42,
      "text": "Renewable energy sources include...",
      "score": 0.87,
      "source": "docs/renewable_energy.md",
      "rank": 1
    }
  ],
  "total_chunks_retrieved": 5,
  "metadata": {
    "model_info": {
      "llm_model": "Phi-4-mini-instruct-Q4_K_M",
      "embedding_model": "granite-embedding-30m-english"
    },
    "latency_ms": {
      "embedding_ms": 50,
      "retrieval_ms": 120,
      "generation_ms": 800,
      "total_ms": 970
    }
  }
}
```

**Retrieval only** (no LLM generation):
```bash
curl -X POST http://localhost:8000/retrieve \
  -H "Content-Type: application/json" \
  -d '{
    "query": "wind energy",
    "max_chunks": 10,
    "query_mode": "hybrid"
  }'
```

**Database statistics**:
```bash
curl http://localhost:8000/stats
```

---

## Programmatic Access

### Python Client Example

```python
#!/usr/bin/env python3
"""Example RAG client"""
import requests

RAG_SERVICE_URL = "http://localhost:8000"

def query_rag(question, max_chunks=5):
    """Submit query to RAG service"""
    response = requests.post(
        f"{RAG_SERVICE_URL}/query",
        json={
            "query": question,
            "max_chunks": max_chunks,
            "query_mode": "hybrid",
            "include_sources": True
        }
    )
    response.raise_for_status()
    return response.json()

# Usage
result = query_rag("What is renewable energy?")
print(f"Answer: {result['answer']}\n")
print(f"Sources ({result['total_chunks_retrieved']}):")
for source in result['sources']:
    print(f"  [{source['rank']}] {source['source']} (score: {source['score']:.3f})")
```

### Bash Script Example

```bash
#!/bin/bash
# query-demo.sh

QUERY="${1:-What is renewable energy?}"

curl -s -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"$QUERY\"}" | jq '.answer'
```

Usage:
```bash
./scripts/query-demo.sh "How does solar energy work?"
```

---

## Performance Expectations

### Query Latency
- **Embedding generation**: 50-100ms
- **Vector retrieval**: 100-200ms
- **LLM generation**: 500-1500ms (depends on answer length)
- **Total**: 1-2 seconds for typical queries

### Concurrent Queries
- **Target**: 10 concurrent queries without degradation
- **Connection pool**: 10 base connections, 20 overflow

### Resource Usage
- **Database**: ~500MB RAM, ~200MB disk (for sample dataset)
- **Ramalama**: 8-12GB RAM (model in memory), 5GB disk (cached model)
- **RAG Service**: 1-2GB RAM
- **Web UI**: 256-512MB RAM

---

## Common Issues and Solutions

### Issue 1: Port Already in Use

**Symptom**:
```
Error: failed to expose ports: requested port 8000 is already allocated
```

**Solution**:
```bash
# Find and kill process using port
lsof -ti :8000 | xargs kill -9

# Or change port in pod.yaml
sed -i 's/hostPort: 8000/hostPort: 8001/' pod/pod.yaml
```

---

### Issue 2: Model Download Fails

**Symptom**:
```
Error downloading model: connection timeout
```

**Solution**:
```bash
# Pre-download model manually
podman run --rm -v ramalama-models-pvc:/models \
  quay.io/ramalama/ramalama:latest \
  pull ollama://Phi-4-mini-instruct-Q4_K_M

# Then deploy pod
podman kube play ./pod/pod.yaml
```

---

### Issue 3: Database Connection Failed

**Symptom**:
```
RAG service logs: psycopg2.OperationalError: could not connect to server
```

**Solution**:
```bash
# Check database is running
podman ps | grep pgvector

# Check database logs
podman logs llamaindex-rag-demo-pgvector

# Verify database initialized
podman exec llamaindex-rag-demo-pgvector \
  psql -U postgres -d ragdb -c "\dt"
# Should list tables including data_documents

# Restart RAG service
podman restart llamaindex-rag-demo-rag-service
```

---

### Issue 4: Out of Memory

**Symptom**:
```
Ramalama container killed (OOMKilled)
```

**Solution**:
```bash
# Use smaller quantized model (edit pod.yaml)
# Change: MODEL_NAME=Phi-4-mini-instruct-Q4_K_M  # 4-bit quantization
# Or: MODEL_NAME=tinyllama  # Much smaller (637MB)

# Reduce context size
# Add env var: RAMALAMA_CONTEXT_SIZE=2048  # Default is 4096

# Or add more RAM to system/VM
```

---

### Issue 5: Slow Query Responses

**Symptom**: Queries taking >10 seconds

**Solution**:
```bash
# Check database index exists
podman exec llamaindex-rag-demo-pgvector \
  psql -U postgres -d ragdb -c "\d data_documents"
# Should show idx_documents_embedding (hnsw)

# Rebuild index if missing
podman exec llamaindex-rag-demo-pgvector \
  psql -U postgres -d ragdb -c "
  CREATE INDEX idx_documents_embedding ON data_documents
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);
"

# Tune HNSW search parameters (edit rag-service config)
# Increase hnsw_ef_search for better recall (slower)
# Decrease for faster search (lower recall)
```

---

## Stopping and Cleanup

### Stop Pod
```bash
# Stop pod (preserves volumes)
podman kube down ./pod/pod.yaml

# Or with Docker Compose
docker-compose down
```

### Remove Volumes
```bash
# CAUTION: This deletes all data

# Remove all volumes
podman volume rm pgvector-pvc ramalama-models-pvc

# Or remove all unused volumes
podman volume prune -f
```

### Complete Cleanup
```bash
# Stop and remove everything
make clean

# Or manually:
podman kube down ./pod/pod.yaml
podman pod rm llamaindex-rag-demo --force
podman volume rm pgvector-pvc ramalama-models-pvc --force
podman rmi localhost/rag-pgvector:latest \
           localhost/ramalama-model:latest \
           localhost/llamaindex-rag:latest \
           localhost/rag-ui:latest
```

---

## Next Steps

After successfully deploying the demo:

1. **Customize the dataset**: Replace `ragdb_dump.sql` with your own docs2db-generated database
2. **Tune retrieval**: Adjust `similarity_threshold`, `max_chunks`, and `query_mode` in queries
3. **Try different models**: Swap out Phi-4 for other models (edit pod.yaml `MODEL_NAME`)
4. **Monitor performance**: Use `/stats` endpoint to track query patterns
5. **Scale up**: Deploy to Kubernetes for production use

---

## Additional Resources

- **API Documentation**: `contracts/rag-service-api.yaml` (OpenAPI spec)
- **Data Model**: `data-model.md` (entity definitions)
- **Architecture**: `research.md` (design decisions)
- **Full Deployment Guide**: `../docs/DEPLOYMENT.md` (detailed instructions)
- **Troubleshooting**: `../docs/TROUBLESHOOTING.md` (issue resolution)

---

## Support and Feedback

For issues or questions:
1. Check troubleshooting section above
2. Review logs: `podman logs <container-name>`
3. Verify health checks: `./scripts/health-check.sh`
4. Open issue on GitHub repository

**Demo complete!** You now have a fully functional RAG system running locally.
