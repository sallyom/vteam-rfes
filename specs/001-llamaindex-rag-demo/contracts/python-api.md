# Python API Contract

**Feature**: LlamaIndex + pgvectordb RAG Demo
**Version**: 1.0.0
**Date**: 2025-10-21

This document specifies the Python API contracts for programmatic access to RAG demo components.

---

## 1. LLM Factory Module

### Module: `llm_factory`

Factory for creating LLM and embedding model instances from configuration.

---

#### Class: `LLMFactory`

**Purpose**: Create LLM and embedding model instances based on YAML configuration.

##### Method: `load_config`

Load configuration from YAML file.

```python
@staticmethod
def load_config(config_path: str = "config.yaml") -> dict
```

**Parameters**:
- `config_path` (str): Path to configuration file. Default: "config.yaml"

**Returns**:
- `dict`: Parsed configuration dictionary

**Raises**:
- `FileNotFoundError`: Config file not found
- `yaml.YAMLError`: Invalid YAML syntax
- `ValueError`: Invalid configuration structure

**Example**:
```python
from llm_factory import LLMFactory

config = LLMFactory.load_config("config.yaml")
print(config['llm']['provider'])  # "ollama"
```

---

##### Method: `create_llm`

Create LLM instance based on provider configuration.

```python
@staticmethod
def create_llm(config: dict) -> BaseLLM
```

**Parameters**:
- `config` (dict): Configuration dictionary (from `load_config()`)

**Returns**:
- `BaseLLM`: LlamaIndex LLM instance (Ollama, OpenAI, or Anthropic)

**Raises**:
- `ValueError`: Unknown provider or missing API key
- `ImportError`: Provider library not installed

**Example**:
```python
config = LLMFactory.load_config()
llm = LLMFactory.create_llm(config)

# Use with LlamaIndex
from llama_index.core import VectorStoreIndex
query_engine = index.as_query_engine(llm=llm)
```

**Provider-Specific Behavior**:
- **Ollama**: Creates `Ollama` instance, validates base_url connectivity
- **OpenAI**: Requires `OPENAI_API_KEY` environment variable
- **Anthropic**: Requires `ANTHROPIC_API_KEY` environment variable, sets tokenizer

---

##### Method: `create_embedding_model`

Create embedding model instance based on configuration.

```python
@staticmethod
def create_embedding_model(config: dict) -> BaseEmbedding
```

**Parameters**:
- `config` (dict): Configuration dictionary (from `load_config()`)

**Returns**:
- `BaseEmbedding`: LlamaIndex embedding model instance

**Raises**:
- `ValueError`: Unknown embedding provider
- `ImportError`: Provider library not installed

**Example**:
```python
config = LLMFactory.load_config()
embed_model = LLMFactory.create_embedding_model(config)

# Verify dimensions
print(embed_model.embed_dim)  # 384
```

---

##### Method: `create_from_config`

Create both LLM and embedding model from configuration file.

```python
@classmethod
def create_from_config(cls, config_path: str = "config.yaml") -> Tuple[BaseLLM, BaseEmbedding]
```

**Parameters**:
- `config_path` (str): Path to configuration file. Default: "config.yaml"

**Returns**:
- `Tuple[BaseLLM, BaseEmbedding]`: (LLM instance, embedding model instance)

**Raises**:
- `FileNotFoundError`: Config file not found
- `ValueError`: Invalid configuration or missing API keys

**Example**:
```python
from llm_factory import LLMFactory

# One-line initialization
llm, embed_model = LLMFactory.create_from_config("config.yaml")

# Use with LlamaIndex Settings
from llama_index.core import Settings
Settings.llm = llm
Settings.embed_model = embed_model
```

---

## 2. Query Engine Module

### Module: `query_engine`

High-level interface for executing RAG queries.

---

#### Class: `RAGQueryEngine`

**Purpose**: Execute queries against pgvector database using LlamaIndex.

##### Constructor: `__init__`

```python
def __init__(
    self,
    vector_store: PGVectorStore,
    llm: Optional[BaseLLM] = None,
    embed_model: Optional[BaseEmbedding] = None,
    top_k: int = 5,
    similarity_threshold: float = 0.5
)
```

**Parameters**:
- `vector_store` (PGVectorStore): Configured pgvector store
- `llm` (BaseLLM, optional): LLM instance. Uses `Settings.llm` if None
- `embed_model` (BaseEmbedding, optional): Embedding model. Uses `Settings.embed_model` if None
- `top_k` (int): Number of chunks to retrieve. Default: 5
- `similarity_threshold` (float): Minimum similarity score (0-1). Default: 0.5

**Example**:
```python
from query_engine import RAGQueryEngine
from llama_index.vector_stores.postgres import PGVectorStore

vector_store = PGVectorStore.from_params(
    database="ragdb",
    host="localhost",
    table_name="llamaindex",
    embed_dim=384
)

query_engine = RAGQueryEngine(
    vector_store=vector_store,
    top_k=5,
    similarity_threshold=0.5
)
```

---

##### Method: `query`

Execute query and return answer with sources.

```python
def query(self, query_text: str, explain: bool = False) -> QueryResult
```

**Parameters**:
- `query_text` (str): User's natural language question
- `explain` (bool): Include detailed retrieval information. Default: False

**Returns**:
- `QueryResult`: Dataclass containing answer, sources, and metadata

**Raises**:
- `ValueError`: Empty query text
- `DatabaseError`: Database connection failed
- `EmbeddingError`: Embedding generation failed
- `LLMError`: LLM generation failed

**Example**:
```python
result = query_engine.query("What is pgvector?")

print(result.answer)
# "pgvector is a PostgreSQL extension..."

for source in result.sources:
    print(f"{source.document} (similarity: {source.similarity_score})")
    # "pgvector_intro.pdf (similarity: 0.87)"
```

---

##### Method: `query_batch`

Execute multiple queries in batch.

```python
def query_batch(self, queries: List[str], parallel: bool = True) -> List[QueryResult]
```

**Parameters**:
- `queries` (List[str]): List of query texts
- `parallel` (bool): Execute queries in parallel. Default: True

**Returns**:
- `List[QueryResult]`: List of query results (same order as input)

**Example**:
```python
queries = [
    "What is pgvector?",
    "How does vector search work?",
    "What are embeddings?"
]

results = query_engine.query_batch(queries)

for query, result in zip(queries, results):
    print(f"Q: {query}")
    print(f"A: {result.answer}\n")
```

---

#### Dataclass: `QueryResult`

Result of a RAG query.

```python
@dataclass
class QueryResult:
    query: str                          # Original query text
    answer: str                         # Generated answer
    sources: List[SourceChunk]          # Retrieved source chunks
    llm_provider: str                   # LLM provider name
    llm_model: str                      # LLM model name
    embedding_model: str                # Embedding model name
    query_time_seconds: float           # Total query time
    retrieval_time_seconds: float       # Vector search time
    generation_time_seconds: float      # LLM generation time
    retrieval_count: int                # Number of chunks retrieved
    metadata: Dict[str, Any]            # Additional metadata
```

**Example**:
```python
result = query_engine.query("What is pgvector?")

print(f"Answer: {result.answer}")
print(f"Sources: {len(result.sources)}")
print(f"Time: {result.query_time_seconds:.2f}s")
print(f"LLM: {result.llm_provider}/{result.llm_model}")
```

---

#### Dataclass: `SourceChunk`

Retrieved source chunk with metadata.

```python
@dataclass
class SourceChunk:
    document_path: str                  # Source document path
    document_filename: str              # Source document name
    chunk_index: int                    # Position in document
    chunk_text: str                     # Chunk content
    similarity_score: float             # Cosine similarity (0-1)
    metadata: Dict[str, Any]            # Additional chunk metadata
```

**Example**:
```python
for source in result.sources:
    print(f"[{source.chunk_index}] {source.document_filename}")
    print(f"  Similarity: {source.similarity_score:.2f}")
    print(f"  Text: {source.chunk_text[:100]}...")
```

---

## 3. Database Utilities Module

### Module: `database_utils`

Utilities for database operations and health checks.

---

#### Function: `create_llamaindex_view`

Create PostgreSQL view for LlamaIndex compatibility.

```python
def create_llamaindex_view(
    connection_string: str,
    view_name: str = "data_llamaindex",
    embedding_model: str = "sentence-transformers/all-MiniLM-L6-v2"
) -> bool
```

**Parameters**:
- `connection_string` (str): PostgreSQL connection string
- `view_name` (str): Name of view to create. Default: "data_llamaindex"
- `embedding_model` (str): Filter to specific embedding model

**Returns**:
- `bool`: True if view created successfully

**Raises**:
- `psycopg2.Error`: Database error

**Example**:
```python
from database_utils import create_llamaindex_view

success = create_llamaindex_view(
    "postgresql://postgres:password@localhost:5432/ragdb",
    view_name="data_llamaindex",
    embedding_model="sentence-transformers/all-MiniLM-L6-v2"
)

if success:
    print("View created successfully")
```

---

#### Function: `verify_schema_compatibility`

Verify docs2db schema is compatible with LlamaIndex.

```python
def verify_schema_compatibility(connection_string: str) -> Dict[str, bool]
```

**Parameters**:
- `connection_string` (str): PostgreSQL connection string

**Returns**:
- `Dict[str, bool]`: Compatibility check results
  ```python
  {
      "documents_table_exists": True,
      "chunks_table_exists": True,
      "embeddings_table_exists": True,
      "models_table_exists": True,
      "pgvector_extension_installed": True,
      "view_exists": True,
      "embedding_dimensions_match": True
  }
  ```

**Example**:
```python
from database_utils import verify_schema_compatibility

checks = verify_schema_compatibility(
    "postgresql://postgres:password@localhost:5432/ragdb"
)

if all(checks.values()):
    print("Schema is compatible")
else:
    failed = [k for k, v in checks.items() if not v]
    print(f"Failed checks: {failed}")
```

---

#### Function: `get_database_stats`

Get statistics about ingested documents and embeddings.

```python
def get_database_stats(connection_string: str) -> DatabaseStats
```

**Parameters**:
- `connection_string` (str): PostgreSQL connection string

**Returns**:
- `DatabaseStats`: Database statistics dataclass

**Example**:
```python
from database_utils import get_database_stats

stats = get_database_stats("postgresql://postgres:password@localhost:5432/ragdb")

print(f"Documents: {stats.document_count}")
print(f"Chunks: {stats.chunk_count}")
print(f"Embeddings: {stats.embedding_count}")
print(f"Database size: {stats.size_mb:.2f} MB")
```

---

#### Dataclass: `DatabaseStats`

Database statistics.

```python
@dataclass
class DatabaseStats:
    document_count: int                 # Total documents
    chunk_count: int                    # Total chunks
    embedding_count: int                # Total embeddings
    model_count: int                    # Embedding models used
    size_mb: float                      # Database size in MB
    avg_chunk_length: float             # Average characters per chunk
    embedding_dimensions: int           # Vector dimensions
    models: List[str]                   # List of embedding model names
```

---

## 4. Health Check Module

### Module: `health_check`

System health verification utilities.

---

#### Function: `check_system_health`

Perform comprehensive system health check.

```python
def check_system_health(config_path: str = "config.yaml") -> HealthCheckResult
```

**Parameters**:
- `config_path` (str): Path to configuration file. Default: "config.yaml"

**Returns**:
- `HealthCheckResult`: Health check results

**Example**:
```python
from health_check import check_system_health

health = check_system_health("config.yaml")

if health.all_passed:
    print("System is healthy")
else:
    for check in health.failed_checks:
        print(f"FAILED: {check.name} - {check.error}")
```

---

#### Dataclass: `HealthCheckResult`

Health check results.

```python
@dataclass
class HealthCheckResult:
    all_passed: bool                    # True if all checks passed
    checks: List[HealthCheck]           # Individual check results
    failed_checks: List[HealthCheck]    # Only failed checks
    timestamp: datetime                 # Check execution time
```

---

#### Dataclass: `HealthCheck`

Individual health check result.

```python
@dataclass
class HealthCheck:
    name: str                           # Check name
    passed: bool                        # Check result
    message: str                        # Status message
    error: Optional[str]                # Error message if failed
    duration_seconds: float             # Check execution time
```

**Check Types**:
- `config_file`: Configuration file exists and is valid
- `database_connection`: Can connect to PostgreSQL
- `database_view`: LlamaIndex view exists and has data
- `embedding_model`: Embedding model can be loaded
- `llm_service`: LLM service is reachable
- `vector_dimensions`: Database dimensions match embedding model

---

## 5. Configuration Module

### Module: `config`

Configuration loading and validation.

---

#### Function: `load_and_validate_config`

Load configuration file and validate structure.

```python
def load_and_validate_config(config_path: str = "config.yaml") -> ValidatedConfig
```

**Parameters**:
- `config_path` (str): Path to configuration file

**Returns**:
- `ValidatedConfig`: Validated configuration object

**Raises**:
- `ConfigurationError`: Invalid configuration structure
- `FileNotFoundError`: Config file not found

**Example**:
```python
from config import load_and_validate_config

config = load_and_validate_config("config.yaml")

print(config.llm.provider)  # "ollama"
print(config.retrieval.top_k)  # 5
```

---

#### Dataclass: `ValidatedConfig`

Validated configuration with typed fields.

```python
@dataclass
class ValidatedConfig:
    llm: LLMConfig                      # LLM configuration
    embedding: EmbeddingConfig          # Embedding configuration
    retrieval: RetrievalConfig          # Retrieval parameters
    database: DatabaseConfig            # Database configuration
```

---

## 6. Usage Examples

### Complete RAG Query Example

```python
from llm_factory import LLMFactory
from query_engine import RAGQueryEngine
from llama_index.vector_stores.postgres import PGVectorStore
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Initialize from configuration
llm, embed_model = LLMFactory.create_from_config("config.yaml")

# Create vector store
vector_store = PGVectorStore.from_params(
    database="ragdb",
    host="localhost",
    table_name="llamaindex",
    embed_dim=384,
    perform_setup=False
)

# Create query engine
query_engine = RAGQueryEngine(
    vector_store=vector_store,
    llm=llm,
    embed_model=embed_model,
    top_k=5,
    similarity_threshold=0.5
)

# Execute query
result = query_engine.query("What is pgvector?")

# Display results
print(f"Answer: {result.answer}\n")
print(f"Sources ({len(result.sources)}):")
for i, source in enumerate(result.sources, 1):
    print(f"  [{i}] {source.document_filename} (similarity: {source.similarity_score:.2f})")

print(f"\nQuery time: {result.query_time_seconds:.2f}s")
```

---

### Health Check Example

```python
from health_check import check_system_health

# Perform health check
health = check_system_health("config.yaml")

# Display results
for check in health.checks:
    status = "✓" if check.passed else "✗"
    print(f"{status} {check.name}: {check.message}")

if not health.all_passed:
    print(f"\n{len(health.failed_checks)} checks failed")
    exit(1)
```

---

### Batch Query Example

```python
from query_engine import RAGQueryEngine
# ... (setup query_engine as above)

queries = [
    "What is pgvector?",
    "How does embedding work?",
    "What databases support vectors?"
]

results = query_engine.query_batch(queries, parallel=True)

for query, result in zip(queries, results):
    print(f"Q: {query}")
    print(f"A: {result.answer}")
    print(f"Time: {result.query_time_seconds:.2f}s\n")
```

---

### Database Statistics Example

```python
from database_utils import get_database_stats

stats = get_database_stats("postgresql://postgres:password@localhost:5432/ragdb")

print(f"Database Statistics")
print(f"===================")
print(f"Documents: {stats.document_count}")
print(f"Chunks: {stats.chunk_count}")
print(f"Embeddings: {stats.embedding_count}")
print(f"Database size: {stats.size_mb:.2f} MB")
print(f"Average chunk length: {stats.avg_chunk_length:.0f} characters")
print(f"Embedding dimensions: {stats.embedding_dimensions}")
print(f"Models: {', '.join(stats.models)}")
```

---

## 7. Type Hints

All functions and methods include complete type hints for IDE autocomplete and type checking.

**Type Checking**:
```bash
mypy query_engine.py llm_factory.py health_check.py
```

---

## 8. Exception Hierarchy

```
RAGException (base)
├── ConfigurationError
│   ├── InvalidConfigError
│   └── MissingAPIKeyError
├── DatabaseError
│   ├── ConnectionError
│   └── SchemaError
├── EmbeddingError
│   ├── ModelNotFoundError
│   └── DimensionMismatchError
└── LLMError
    ├── ServiceUnavailableError
    └── GenerationError
```

---

## 9. Testing

All public APIs include docstring examples that serve as doctests.

**Run doctests**:
```bash
python -m doctest query_engine.py llm_factory.py -v
```

---

This Python API provides programmatic access to all RAG demo functionality for integration into larger applications or custom workflows.
