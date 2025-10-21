# Research: LlamaIndex + pgvectordb RAG Demo

**Date**: 2025-10-21
**Feature**: LlamaIndex + pgvectordb RAG Demo
**Branch**: `001-llamaindex-rag-demo`

This document captures research findings for key technical decisions in the implementation plan.

---

## 1. Schema Compatibility: docs2db vs LlamaIndex PGVectorStore

### Decision
**INCOMPATIBLE - Database View Adapter Required**

docs2db and LlamaIndex use fundamentally different database schemas. We will create a PostgreSQL VIEW that adapts docs2db's normalized schema to LlamaIndex's denormalized expectations.

### Rationale

**docs2db Schema (Normalized)**:
- 4 tables: `documents`, `chunks`, `embeddings`, `models`
- Foreign key relationships between tables
- `embeddings.embedding` stored in separate table with model tracking

**LlamaIndex PGVectorStore Schema (Denormalized)**:
- Single table: `data_{table_name}`
- Columns: `id`, `text`, `node_id`, `embedding`, `metadata_`
- All data in one table for fast retrieval

**Key Incompatibilities**:
| Aspect | docs2db | LlamaIndex | Compatible? |
|--------|---------|------------|-------------|
| Table Structure | 4 normalized tables | Single table | ❌ NO |
| Vector Storage | `embeddings.embedding` | `embedding` column | ❌ NO |
| Text Content | `chunks.text` | `text` column | ❌ NO |
| Metadata | `chunks.metadata` (JSONB) | `metadata_` (JSONB) | ⚠️ Different location |
| Node ID | `chunks.id` (integer) | `node_id` (string) | ❌ NO |

### Alternatives Considered

1. **Database View (Recommended)**: Create VIEW that joins docs2db tables and presents them as LlamaIndex schema
   - Pros: No data duplication, real-time sync, read-only protection
   - Cons: Read-only (LlamaIndex cannot write back), slight query overhead

2. **ETL Pipeline**: Sync data from docs2db to LlamaIndex table
   - Pros: Full read/write, optimized for LlamaIndex
   - Cons: Data duplication, sync lag, more maintenance

3. **Custom Vector Store**: Implement custom LlamaIndex vector store for docs2db
   - Pros: Most flexible, maintains normalized schema
   - Cons: Significant development effort, ongoing maintenance

4. **One-time Migration**: Migrate docs2db data to LlamaIndex format
   - Pros: Native LlamaIndex performance
   - Cons: Loses docs2db multi-model capability, cannot easily update

### Implementation Notes

**PostgreSQL View Creation**:
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
        'model_name', m.name,
        'model_dimensions', m.dimensions
    )::JSONB as metadata_
FROM embeddings e
JOIN chunks c ON e.chunk_id = c.id
JOIN documents d ON c.document_id = d.id
JOIN models m ON e.model_id = m.id
WHERE m.name = 'sentence-transformers/all-MiniLM-L6-v2'
ORDER BY e.id;
```

**LlamaIndex Configuration**:
```python
from llama_index.vector_stores.postgres import PGVectorStore

vector_store = PGVectorStore.from_params(
    database="ragdb",
    table_name="llamaindex",  # Will query data_llamaindex view
    embed_dim=384,            # MUST match all-MiniLM-L6-v2
    perform_setup=False,      # Don't let LlamaIndex create tables
)
```

**Critical Requirements**:
- Embedding dimensions must match (384 for all-MiniLM-L6-v2)
- Same embedding model used in docs2db and LlamaIndex
- View must be created as part of database dump preparation
- Include view creation in demo's initialization scripts

---

## 2. Chunking Strategy and Embedding Model Configuration

### Decision
**Chunk Size**: 512 tokens
**Chunk Overlap**: 102 tokens (20%)
**Embedding Model**: sentence-transformers/all-MiniLM-L6-v2 (384 dimensions)
**Splitting Method**: LlamaIndex SentenceSplitter with sentence boundaries

### Rationale

**Chunk Size (512 tokens)**:
- Matches docs2db's existing configuration (`.../docs2db/src/docs2db/const.py`)
- Balances context capture with embedding quality
- Research shows 512-1024 tokens optimal for QA tasks
- all-MiniLM-L6-v2 supports up to 512 tokens (trained on 128, default max 256)
- Trade-off: Slightly exceeds optimal 256-token range but captures more context

**Chunk Overlap (20%)**:
- Industry best practice: 10-20% overlap
- 102 tokens preserves context across chunk boundaries
- Prevents information loss for concepts spanning multiple chunks
- Small storage cost (~20% more chunks) for significant quality improvement

**Embedding Model (all-MiniLM-L6-v2)**:
- Already used by docs2db (consistency requirement)
- 384 dimensions: optimal balance of quality and efficiency
- 22MB model size: fast inference, low memory footprint
- 84-85% accuracy on STS-B semantic similarity benchmarks
- Production-ready: widely adopted in RAG systems

**Critical Consistency Requirement**:
- MUST use identical embedding model for ingestion and query time
- Different models create incompatible vector spaces → poor retrieval
- docs2db uses: `sentence-transformers/all-MiniLM-L6-v2` (384d)
- LlamaIndex MUST use: `sentence-transformers/all-MiniLM-L6-v2` (384d)

### Alternatives Considered

**256-token chunks**:
- Pros: Optimal for all-MiniLM-L6-v2 (within training range)
- Cons: Requires re-chunking all docs2db content, less context per chunk

**1024-token chunks**:
- Pros: Maximum context, fewer chunks
- Cons: 4x over model optimal range, truncation loses 75% of content

**all-mpnet-base-v2 embedding model**:
- Pros: Better accuracy (88% vs 85%), 768 dimensions
- Cons: 2-3x slower, 420MB model size, requires full re-ingestion

**bge-small-en-v1.5**:
- Pros: Strong retrieval, 384 dimensions
- Cons: Breaking change, requires full re-ingestion

### Implementation Notes

**docs2db Configuration (already optimal)**:
```python
# /workspace/.../docs2db/src/docs2db/const.py
CHUNKING_CONFIG = {
    "max_tokens": 512,
    "merge_peers": True,
    "chunker_class": "HybridChunker",
    "tokenizer_model": "sentence-transformers/all-MiniLM-L6-v2",
}
```

**LlamaIndex Configuration**:
```python
from llama_index.core.node_parser import SentenceSplitter
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

# Chunking
node_parser = SentenceSplitter(
    chunk_size=512,        # Match docs2db
    chunk_overlap=102,     # 20% overlap
    separator=" ",
    paragraph_separator="\n\n",
)

# Embedding model (MUST match docs2db)
embed_model = HuggingFaceEmbedding(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    device="cpu",  # or "cuda"
)

# Global settings
from llama_index.core import Settings
Settings.chunk_size = 512
Settings.chunk_overlap = 102
Settings.embed_model = embed_model
Settings.node_parser = node_parser
```

**Validation Checklist**:
- [ ] Verify vector dimensions match (384 for both systems)
- [ ] Confirm same embedding model in docs2db and LlamaIndex
- [ ] Test queries return similarity scores > 0.5
- [ ] Validate metadata accessible from LlamaIndex
- [ ] Check performance with realistic dataset

---

## 3. Multi-LLM Provider Architecture

### Decision
**Factory Pattern with Configuration-Driven Provider Initialization**

Implement a factory pattern that creates LLM instances based on YAML configuration, using LlamaIndex's Settings singleton for global defaults while supporting local overrides.

### Rationale

**Architecture Components**:
1. Configuration file (YAML) specifying provider type and parameters
2. Factory function/class to instantiate correct LLM provider
3. Environment variables for API keys (never commit secrets)
4. LlamaIndex Settings for global defaults with local override capability

**Benefits**:
- **Maintainability**: Single config file, centralized initialization logic
- **User Experience**: Switch providers by editing YAML + environment variables, no code changes
- **LlamaIndex Native**: Leverages Settings singleton and provider abstraction
- **Production Ready**: 12-factor app principles, thread-safe local configuration

### Alternatives Considered

**Direct Settings Modification**:
- Pros: Simplest, minimal abstraction
- Cons: Requires code changes, configuration scattered

**LiteLLM Proxy Layer**:
- Pros: Unified API, built-in retry/fallback
- Cons: Additional dependency, may not expose all features, overkill

**Strategy Pattern with Explicit Provider Classes**:
- Pros: Full control, extensible, clean OOP
- Cons: More boilerplate, over-engineered for simple switching

### Implementation Notes

**Configuration File Structure (config.yaml)**:
```yaml
llm:
  provider: "ollama"  # Options: ollama, openai, anthropic

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

embedding:
  provider: "huggingface"

  huggingface:
    model_name: "sentence-transformers/all-MiniLM-L6-v2"
    embed_dim: 384

retrieval:
  top_k: 5
  similarity_threshold: 0.5
```

**Environment Variables (.env)**:
```bash
# Never commit to source control!
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...

# Database
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=ragdb
POSTGRES_PASSWORD=password
```

**Factory Implementation (llm_factory.py)**:
```python
import os
import yaml
from typing import Tuple
from llama_index.core.llms import BaseLLM
from llama_index.core.embeddings import BaseEmbedding

class LLMFactory:
    @staticmethod
    def load_config(config_path: str = "config.yaml") -> dict:
        with open(config_path, 'r') as f:
            return yaml.safe_load(f)

    @staticmethod
    def create_llm(config: dict) -> BaseLLM:
        provider = config['llm']['provider']

        if provider == "ollama":
            from llama_index.llms.ollama import Ollama
            return Ollama(**config['llm']['ollama'])

        elif provider == "openai":
            from llama_index.llms.openai import OpenAI
            params = config['llm']['openai'].copy()
            params['api_key'] = os.getenv('OPENAI_API_KEY')
            if not params['api_key']:
                raise ValueError("OPENAI_API_KEY not set")
            return OpenAI(**params)

        elif provider == "anthropic":
            from llama_index.llms.anthropic import Anthropic
            from llama_index.core import Settings
            params = config['llm']['anthropic'].copy()
            params['api_key'] = os.getenv('ANTHROPIC_API_KEY')
            if not params['api_key']:
                raise ValueError("ANTHROPIC_API_KEY not set")
            llm = Anthropic(**params)
            Settings.tokenizer = llm.tokenizer  # CRITICAL
            return llm

        raise ValueError(f"Unknown provider: {provider}")

    @staticmethod
    def create_embedding_model(config: dict) -> BaseEmbedding:
        provider = config['embedding']['provider']

        if provider == "huggingface":
            from llama_index.embeddings.huggingface import HuggingFaceEmbedding
            return HuggingFaceEmbedding(
                model_name=config['embedding']['huggingface']['model_name']
            )

        raise ValueError(f"Unknown provider: {provider}")
```

**Usage in Application**:
```python
from llm_factory import LLMFactory
from llama_index.core import Settings
from dotenv import load_dotenv

load_dotenv()  # Load environment variables

factory = LLMFactory()
config = factory.load_config("config.yaml")

llm = factory.create_llm(config)
embed_model = factory.create_embedding_model(config)

# Set global defaults
Settings.llm = llm
Settings.embed_model = embed_model
```

**Provider-Specific Initialization**:

**Ollama**:
```python
from llama_index.llms.ollama import Ollama

llm = Ollama(
    model="llama3.1:latest",
    request_timeout=120.0,  # Default 30s often insufficient
    context_window=8000,
    base_url="http://localhost:11434",
    temperature=0.7,
)
```
- Installation: `pip install llama-index-llms-ollama`
- Requires Ollama running locally
- First use downloads model (can be GBs)

**OpenAI**:
```python
from llama_index.llms.openai import OpenAI

llm = OpenAI(
    model="gpt-4o-mini",
    temperature=0.7,
    max_tokens=512,
    api_key=os.getenv('OPENAI_API_KEY'),
)
```
- Installation: `pip install llama-index-llms-openai`
- Requires `OPENAI_API_KEY` environment variable
- API calls cost money

**Anthropic**:
```python
from llama_index.llms.anthropic import Anthropic
from llama_index.core import Settings

llm = Anthropic(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    temperature=1.0,
    api_key=os.getenv('ANTHROPIC_API_KEY'),
)

# CRITICAL: Set tokenizer for accurate token counting
Settings.tokenizer = llm.tokenizer
```
- Installation: `pip install llama-index-llms-anthropic`
- Requires `ANTHROPIC_API_KEY` environment variable
- No embedding model provided (use HuggingFace)

**Key Compatibility Issues**:

1. **Embedding Dimension Consistency (CRITICAL)**:
   - Vector stores are dimension-locked on first write
   - OpenAI embeddings: 1536 dims
   - all-MiniLM-L6-v2: 384 dims
   - Switching requires recreating vector store
   - **Solution**: Use same embedding model always, document in README

2. **Context Window Mismatches**:
   - Llama 3.1: 128K tokens
   - GPT-4: 128K tokens
   - Claude Sonnet: 200K tokens
   - **Solution**: Set `context_window` explicitly or let LlamaIndex auto-detect

3. **Anthropic Tokenizer**:
   - MUST set `Settings.tokenizer = llm.tokenizer`
   - Older models (claude-2.1) no longer supported in latest client

4. **Ollama Timeout**:
   - Default 30s often insufficient
   - Set `request_timeout=120.0` or higher

5. **Threading and Global State**:
   - Settings is a singleton (potential thread issues)
   - **Solution**: Pass LLM directly to components for thread-safety
   ```python
   # Thread-safe
   query_engine = index.as_query_engine(llm=llm)

   # Potentially unsafe with threading
   query_engine = index.as_query_engine()  # Uses global Settings.llm
   ```

**Security Best Practices**:
- Never commit API keys to source control
- Add `.env` to `.gitignore`
- Provide `.env.example` template (without actual keys)
- Use environment variables for all secrets
- Support key rotation without app restart

---

## Summary

All three technical unknowns have been resolved:

1. **Schema Compatibility**: Use PostgreSQL VIEW to adapt docs2db schema to LlamaIndex expectations
2. **Chunking/Embeddings**: Use 512-token chunks with 20% overlap, all-MiniLM-L6-v2 embeddings (384d)
3. **Multi-LLM Support**: Implement factory pattern with YAML configuration and environment variables

These decisions enable the demo to:
- Work seamlessly with docs2db-generated databases
- Maintain optimal retrieval quality
- Support multiple LLM providers with simple configuration changes
- Follow production-ready patterns (12-factor app, security best practices)

The implementation can now proceed to Phase 1 (data model and contracts design).
