# Research Findings: Llamaindex RAG Demo with Ramalama

**Date**: 2025-10-31
**Feature**: 001-llamaindex-ramalama-demo
**Purpose**: Resolve technical clarifications for implementation planning

## Executive Summary

This research resolves all "NEEDS CLARIFICATION" items from the Technical Context. Key decisions:
- **Container Orchestration**: Podman pods with Kubernetes YAML (matches llamastack demo)
- **Ramalama**: OpenAI-compatible model serving with llama.cpp runtime
- **Llamaindex + PGVector**: Official integration via `llama-index-vector-stores-postgres`
- **Web UI**: Streamlit (consistent with reference demo)
- **Networking**: Shared localhost (127.0.0.1) within pod, simplified service discovery

---

## 1. Ramalama Model Serving

### Decision
Use **Ramalama** as container-native model serving infrastructure with OpenAI-compatible API endpoints.

### Rationale
- **Container-native design**: Aligns with containerized architecture requirement
- **OpenAI API compatibility**: Drop-in replacement for Llamaindex endpoints
- **Zero host dependencies**: GPU libraries included in containers
- **Security-first**: Rootless containers, read-only mounts, network isolation
- **Matches reference**: Consistent with existing llamastack demo architecture

### Key Capabilities
**Model serving runtimes**:
- llama.cpp (default) - Broad compatibility, CPU/GPU support
- vLLM - High-throughput production workloads
- MLX - Apple Silicon optimization

**API endpoints** (llama.cpp):
- `/v1/chat/completions` - OpenAI-compatible chat interface
- `/v1/embeddings` - OpenAI-compatible embeddings
- `/health` - Health check endpoint

**Hardware support**:
- Automatic GPU detection (NVIDIA, AMD, Intel, Apple Silicon)
- CPU fallback when no GPU present
- GPU-specific container images pulled automatically

### Configuration Requirements
```yaml
ramalama-model:
  image: quay.io/ramalama/ramalama:latest
  ports:
    - containerPort: 8080  # API endpoint
  env:
    - MODEL_NAME: "unsloth/Phi-4-mini-instruct-GGUF/Phi-4-mini-instruct-Q4_K_M.gguf"
  resources:
    requests:
      memory: 12Gi
    limits:
      memory: 16Gi
  volumeMounts:
    - name: model-cache
      mountPath: /var/lib/ramalama  # Persistent model storage
  readinessProbe:
    httpGet:
      path: /health
      port: 8080
    initialDelaySeconds: 30  # Model loading time
```

### Alternatives Considered
- **Ollama**: Simpler but less container-native, no built-in RAG
- **LocalAI**: Similar goals but different API patterns
- **Text Generation Inference (TGI)**: HuggingFace solution, heavier dependencies
- **Direct llama.cpp**: More control but higher setup complexity

---

## 2. Llamaindex + PGVector Integration

### Decision
Use **`llama-index-vector-stores-postgres`** package with HNSW indexing for vector similarity search.

### Rationale
- **Official support**: Native PGVector integration by Llamaindex
- **Pre-existing embeddings**: Supports loading from existing vectors without re-embedding
- **Hybrid search**: Combined vector + full-text search capabilities
- **Production-ready**: Connection pooling, async support, metadata filtering
- **Performance**: HNSW indexing for fast approximate nearest neighbor search

### Integration Pattern
```python
from llama_index.vector_stores.postgres import PGVectorStore
from llama_index.core import VectorStoreIndex

# Connect to existing PGVector database
vector_store = PGVectorStore.from_params(
    database="ragdb",
    host="127.0.0.1",  # Shared localhost in pod
    port=5432,
    user="postgres",
    password="password",
    table_name="documents",
    embed_dim=768,  # Match docs2db embeddings
    perform_setup=False,  # Don't modify existing schema
    hybrid_search=True,  # Enable text + vector search
    hnsw_kwargs={
        "hnsw_m": 16,
        "hnsw_ef_construction": 64,
        "hnsw_ef_search": 40
    }
)

# Load index from existing vectors (NO re-embedding)
index = VectorStoreIndex.from_vector_store(
    vector_store=vector_store
)

# Query with Ramalama LLM
from llama_index.llms.openai_like import OpenAILike

llm = OpenAILike(
    api_base="http://127.0.0.1:8080/v1",
    api_key="dummy",
    model="local-model"
)

query_engine = index.as_query_engine(
    llm=llm,
    similarity_top_k=5
)
```

### Key Features
- **No re-embedding**: `VectorStoreIndex.from_vector_store()` uses existing embeddings
- **Metadata filtering**: Rich filtering with operators (in, nin, contains, >=, <=)
- **Query strategies**: Vector similarity, full-text search, hybrid mode
- **Connection pooling**: `pool_size`, `max_overflow` for concurrent queries
- **HNSW vs IVFFlat**: HNSW recommended for <1M vectors

### Configuration Requirements
**Dependencies**:
```bash
pip install llama-index-vector-stores-postgres
pip install psycopg2-binary pgvector asyncpg "sqlalchemy[asyncio]" greenlet
```

**Table Schema** (PGVector):
```sql
CREATE TABLE data_documents (
    id BIGSERIAL PRIMARY KEY,
    node_id VARCHAR,
    text VARCHAR NOT NULL,
    metadata_ JSONB,
    embedding VECTOR(768),  # Match docs2db dimensions
    text_search_tsv TSVECTOR  # For hybrid search
);

CREATE INDEX ON data_documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

### Performance Considerations
- **HNSW indexing**: Default choice for datasets <1M vectors
- **Query latency**: <200ms p95 for vector similarity search
- **Top-K tuning**: Start with 5, adjust based on relevance
- **Hybrid search overhead**: ~2x latency vs pure vector search

---

## 3. Container Orchestration

### Decision
Use **Podman pods** with Kubernetes YAML manifests as primary format, provide docker-compose.yml as alternative.

### Rationale
- **Matches reference demo**: Existing llamastack demo uses Podman pods
- **Simplified networking**: All containers share localhost (127.0.0.1) within pod
- **Kubernetes compatibility**: Pod YAML can be deployed to Kubernetes unchanged
- **Security**: Rootless containers by default
- **Single deployment unit**: All services start/stop together

### Architecture Pattern
```
Podman Pod: llamaindex-rag-demo
├── pgvector (Port 5432)
│   └── PostgreSQL + pgvector extension
├── ramalama-model (Port 8080)
│   └── Model serving (inference + embeddings)
├── rag-service (Port 8000)
│   └── Llamaindex RAG orchestration (FastAPI)
└── web-ui (Port 8501)
    └── Streamlit interface
```

### Networking Configuration
**Podman Pod** (Recommended):
```yaml
spec:
  hostNetwork: false  # Pod has own network namespace
  containers:
  - name: pgvector
    ports:
    - containerPort: 5432
  - name: ramalama
    env:
    - name: MODEL_PORT
      value: "8080"
  - name: rag-service
    env:
    - name: POSTGRES_HOST
      value: "127.0.0.1"  # All services use localhost
    - name: RAMALAMA_URL
      value: "http://127.0.0.1:8080"
  - name: web-ui
    ports:
    - containerPort: 8501
      hostPort: 8501  # Only UI exposed to host
```

**Docker Compose** (Alternative):
```yaml
services:
  db:
    # Internal only - no ports published
  ramalama:
    # Internal only
  rag-service:
    depends_on: [db, ramalama]
    # Internal only
  web-ui:
    depends_on: [rag-service]
    ports: ["8501:8501"]  # Only UI exposed
networks:
  default:
    driver: bridge
```

### Volume Management
**Persistent volumes**:
1. **postgres-data**: Database persistence (survives restarts)
2. **model-cache**: Ramalama model storage (20GB, avoids re-downloads)

**Ephemeral volumes**:
3. **/dev/shm**: Shared memory for ML operations (2GB)

**Configuration**:
4. **ConfigMap**: Service configuration (read-only)

```yaml
volumes:
  - name: postgres-data
    persistentVolumeClaim:
      claimName: pgvector-pvc
  - name: model-cache
    persistentVolumeClaim:
      claimName: ramalama-models-pvc
  - name: shm
    emptyDir:
      medium: Memory
      sizeLimit: 2Gi
  - name: config
    configMap:
      name: rag-demo-config
```

### Health Checks and Startup Ordering
**Database** (PostgreSQL):
```yaml
readinessProbe:
  exec:
    command: ["pg_isready", "-U", "postgres", "-d", "ragdb"]
  initialDelaySeconds: 5
  periodSeconds: 10
```

**Model Server** (Ramalama):
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 30  # 5 minutes for model download/load
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 15
```

**RAG Service** (Llamaindex):
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Deployment Workflow
```bash
# 1. Build images
./scripts/build-containers.sh

# 2. Create persistent volumes
podman kube play - <<EOF
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

# 3. Deploy pod
podman kube play --configmap ./configmap.yaml ./pod.yaml

# 4. Monitor startup
podman logs -f llamaindex-rag-demo-rag-service

# 5. Verify health
curl http://localhost:8000/health
open http://localhost:8501
```

---

## 4. Llamastack Demo Architecture

### Reference Demo Location
`/workspace/sessions/agentic-session-1761932488/workspace/docs2db-api/demos/llama-stack/ramalama/`

### Key Files
- **pod.yaml**: 4-container pod orchestration
- **configmap.yaml**: Llama Stack YAML configuration
- **Containerfile**: PostgreSQL + pgvector image
- **Containerfile.llamastack**: Llama Stack service image
- **entrypoint.sh**: Database initialization script
- **client.py**: Direct RAG tool calling demo
- **agent_tool_calling_client.py**: Agent-based demo

### Architecture Insights
**4-container pattern**:
1. **pgvector**: PostgreSQL with pgvector extension
2. **llama-stack**: RAG orchestration service (analogous to your Llamaindex service)
3. **ramalama-model**: Model serving (inference + embeddings)
4. **ui**: Streamlit web interface

**Configuration approach**:
- ConfigMap for service configuration (YAML format)
- Environment variables for connection strings
- Persistent volumes for models and database
- Health checks on all services

**Database integration**:
- Pre-loaded docs2db dump (ragdb_dump.sql)
- Entrypoint script handles both text and binary format dumps
- PGVector for vector storage (384 dimensions, all-MiniLM-L6-v2)
- Connection pooling via environment variables

**Client patterns**:
- **Direct tool calling**: Explicit invocation of search_documents tool
- **Agent-based**: Autonomous tool selection by agent

### Replication Strategy for Llamaindex Demo
1. **Keep 4-container structure**: Don't reduce to fewer containers
2. **Use Podman pod YAML**: Maintain consistency with reference
3. **ConfigMap pattern**: Separate configuration from images
4. **Shared localhost networking**: All services on 127.0.0.1
5. **Database entrypoint**: Auto-load dump on first run
6. **Persistent model cache**: Avoid re-downloading models
7. **Streamlit UI**: Use same web framework
8. **Dual client examples**: Provide both direct API and higher-level clients

---

## 5. Web UI Framework

### Decision
Use **Streamlit** for web-based interface.

### Rationale
- **Matches reference demo**: Llamastack demo uses Streamlit
- **Rapid development**: Simple Python API for UI creation
- **Built-in components**: Chat interface, file upload, markdown rendering
- **Real-time updates**: Automatic UI refresh on code changes
- **Low resource requirements**: 256-512Mi RAM

### Implementation Pattern
```python
# app.py
import streamlit as st
import requests

st.title("Llamaindex RAG Demo")

# Query input
query = st.text_input("Enter your question:")

if st.button("Search"):
    # Call RAG service
    response = requests.post(
        "http://127.0.0.1:8000/query",
        json={"query": query, "top_k": 5}
    )

    # Display answer
    result = response.json()
    st.write("### Answer")
    st.write(result["answer"])

    # Display sources
    st.write("### Sources")
    for source in result["sources"]:
        with st.expander(f"Score: {source['score']:.3f}"):
            st.write(source["text"])
```

### Container Configuration
```yaml
web-ui:
  image: localhost/llamaindex-ui:latest
  ports:
    - containerPort: 8501
      hostPort: 8501  # Expose to host
  env:
    - name: RAG_SERVICE_URL
      value: "http://127.0.0.1:8000"
  resources:
    requests:
      memory: 256Mi
    limits:
      memory: 512Mi
```

### Alternatives Considered
- **Gradio**: Similar to Streamlit, but less familiar in reference demo
- **React/Vue SPA**: More complex, requires separate build process
- **FastAPI templates**: Less interactive, requires more frontend code

---

## 6. docs2db Database Dump Format

### Format Specification
Based on existing llamastack demo:
- **File name**: `ragdb_dump.sql`
- **Format**: PostgreSQL SQL dump (text or binary format)
- **Contents**: Database schema + data (documents, embeddings, metadata)

### Import Mechanism
**Entrypoint script pattern** (from llamastack demo):
```bash
#!/bin/bash
# entrypoint.sh

# Initialize PostgreSQL
if [ ! -s "/var/lib/postgresql/data/PG_VERSION" ]; then
    initdb -D /var/lib/postgresql/data
    pg_ctl -D /var/lib/postgresql/data -l logfile start

    # Create database
    createdb ragdb

    # Load dump
    if [ -f "/docker-entrypoint-initdb.d/ragdb_dump.sql" ]; then
        echo "Loading database dump..."
        psql -d ragdb < /docker-entrypoint-initdb.d/ragdb_dump.sql
    fi

    pg_ctl -D /var/lib/postgresql/data stop
fi

# Start PostgreSQL
postgres -D /var/lib/postgresql/data
```

### Containerfile Pattern
```dockerfile
FROM pgvector/pgvector:pg17

# Copy database dump
COPY ragdb_dump.sql /docker-entrypoint-initdb.d/

# Copy initialization script
COPY entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/entrypoint.sh

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

### Expected Schema
Based on docs2db and PGVector integration:
```sql
-- Documents table (from docs2db)
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    source VARCHAR,
    content TEXT,
    metadata JSONB
);

-- Embeddings table (PGVector)
CREATE TABLE embeddings (
    id SERIAL PRIMARY KEY,
    document_id INTEGER REFERENCES documents(id),
    chunk_text TEXT,
    embedding VECTOR(768),  # Granite embedding dimensions
    metadata JSONB
);

-- Vector index
CREATE INDEX ON embeddings
USING hnsw (embedding vector_cosine_ops);
```

### Compatibility Notes
- **Embedding dimensions**: Must match Granite model (typically 768)
- **Table naming**: Llamaindex expects `data_<table_name>` prefix
- **Metadata column**: Named `metadata_` (with underscore) in Llamaindex
- **Solution**: Create SQL views to map docs2db schema to Llamaindex expectations

---

## Key Decisions Summary

| Decision Point | Selected Approach | Rationale |
|----------------|-------------------|-----------|
| **Container Orchestration** | Podman pods (Kubernetes YAML) | Matches reference, simpler networking |
| **Model Serving** | Ramalama with llama.cpp | Container-native, OpenAI-compatible |
| **RAG Framework** | Llamaindex with PGVectorStore | Official PGVector integration, pre-existing embeddings support |
| **Vector Database** | PostgreSQL + PGVector + HNSW | Production-ready, fast similarity search |
| **Web UI** | Streamlit | Matches reference, rapid development |
| **Networking** | Shared localhost (127.0.0.1) | Pod network namespace, no service discovery needed |
| **Configuration** | ConfigMap + environment variables | Separates config from images, Kubernetes-compatible |
| **Database Init** | Entrypoint script + dump mount | Auto-load on first run, matches reference |
| **Model Storage** | Persistent volume (20GB) | Avoid re-downloads, faster subsequent starts |

---

## Implementation Recommendations

### Phase 1: Core Services
1. **Database container**: PostgreSQL + PGVector with entrypoint script
2. **Ramalama container**: Model serving with health checks
3. **RAG service container**: Llamaindex + FastAPI with health/ready endpoints
4. **Pod configuration**: Kubernetes YAML with ConfigMap

### Phase 2: User Interface
5. **Streamlit UI**: Interactive query interface
6. **Client examples**: Python scripts for programmatic access

### Phase 3: Documentation
7. **README.md**: Quickstart guide with deployment commands
8. **DEPLOYMENT.md**: Detailed setup and configuration
9. **TROUBLESHOOTING.md**: Common issues and solutions

### Deployment Timeline
- **Build phase**: 5-10 minutes (container images)
- **Model download**: 5-15 minutes (first run only, cached afterward)
- **Initialization**: 2-5 minutes (database + service startup)
- **Total**: 12-30 minutes (15-minute target achievable after first run)

---

## Next Steps

With all clarifications resolved, proceed to Phase 1: Design & Contracts
- Generate data-model.md (entities and relationships)
- Create API contracts (OpenAPI specifications)
- Produce quickstart.md (deployment guide)
- Update agent context with technologies

**Status**: All NEEDS CLARIFICATION items resolved. Ready for Phase 1.
