# Implementation Plan: LlamaIndex + pgvectordb RAG Demo

**Branch**: `001-llamaindex-rag-demo` | **Date**: 2025-10-21 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-llamaindex-rag-demo/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Create a containerized RAG (Retrieval-Augmented Generation) demonstration using LlamaIndex and pgvector database. The demo enables users to ingest documents, create a vector database dump, deploy containers via pod.yaml, and execute queries against their document corpus. The system demonstrates the complete workflow from raw documents to a functioning question-answering system with source attribution.

## Technical Context

**Language/Version**: Python 3.11+
**Primary Dependencies**: LlamaIndex, pgvector, sentence-transformers (all-MiniLM-L6-v2), docs2db, psycopg2/asyncpg
**Storage**: PostgreSQL with pgvector extension for vector embeddings
**Testing**: pytest for Python components, integration tests for container orchestration
**Target Platform**: Container runtime (Podman/Docker) on Linux/macOS
**Project Type**: Demo application with CLI and containerized services
**Performance Goals**: Query response < 5 seconds, database dump load < 5 minutes for typical datasets (1,000-5,000 chunks)
**Constraints**: Total pod memory < 8GB, embedding model compatibility between ingestion and query (all-MiniLM-L6-v2), idempotent database initialization
**Scale/Scope**: Support document sets up to 5,000 chunks (~500MB dump), handle 20+ consecutive queries without degradation

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Note**: Constitution file is currently a template. Proceeding with general best practices:

**Initial Check (Pre-Research)**:
- ✓ Demo provides clear value proposition (end-to-end RAG workflow)
- ✓ Containerized approach ensures reproducibility
- ✓ Documentation-first approach specified in requirements
- ✓ Testing requirements defined (FR-018 verification steps)
- ⚠️ NEEDS CLARIFICATION: Schema compatibility between docs2db and LlamaIndex (FR-009)
- ⚠️ NEEDS CLARIFICATION: Optimal chunking strategy and embedding model configuration
- ⚠️ NEEDS CLARIFICATION: Multi-LLM provider architecture pattern

**Post-Design Check (After Phase 1)**:
- ✓ All clarifications resolved (see research.md)
- ✓ Schema compatibility addressed via PostgreSQL VIEW adapter (research.md §1)
- ✓ Chunking strategy defined: 512 tokens, 20% overlap, all-MiniLM-L6-v2 (research.md §2)
- ✓ Multi-LLM architecture: Factory pattern with YAML configuration (research.md §3)
- ✓ Data model documented with clear entity relationships (data-model.md)
- ✓ API contracts specified for CLI and Python interfaces (contracts/)
- ✓ Quickstart guide provides end-to-end workflow (quickstart.md)
- ✓ Agent context updated with technology stack

**Design Quality Gates**:
- ✓ No architectural violations or unjustified complexity
- ✓ Solution maintains separation of concerns (docs2db layer, LlamaIndex layer, query layer)
- ✓ Read-only view prevents accidental schema modifications
- ✓ Configuration-driven design enables flexibility without code changes
- ✓ Security best practices: environment variables for secrets, no committed keys

## Project Structure

### Documentation (this feature)

```
specs/001-llamaindex-rag-demo/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```
demos/llamaindex-rag-demo/
├── README.md                    # End-to-end user documentation
├── pod.yaml                     # Pod specification for pgvector + LlamaIndex containers
├── config.yaml                  # Configuration for LLM providers, retrieval parameters
├── architecture.md              # Architecture diagram and data flow
├── ingestion/
│   ├── ingest_documents.py      # Document ingestion script using docs2db
│   ├── create_dump.sh           # Database dump creation script
│   └── requirements.txt         # Python dependencies for ingestion
├── pgvector/
│   ├── Dockerfile               # Custom pgvector image with dump loading
│   ├── entrypoint.sh            # Custom entrypoint for idempotent initialization
│   └── init_scripts/            # SQL scripts for schema setup
├── llamaindex-service/
│   ├── Dockerfile               # LlamaIndex service container
│   ├── app.py                   # Main query service
│   ├── query_cli.py             # CLI for executing queries
│   ├── llm_providers.py         # Abstraction for multi-LLM support
│   ├── health_check.py          # Health check endpoints
│   └── requirements.txt         # Python dependencies
├── examples/
│   ├── sample_docs/             # Sample documents for testing
│   └── example_queries.txt      # Example queries demonstrating functionality
└── tests/
    ├── test_ingestion.py        # Tests for document ingestion
    ├── test_query.py            # Tests for query execution
    └── test_integration.py      # End-to-end integration tests
```

**Structure Decision**: Demo application structure with separate directories for ingestion tooling, container definitions (pgvector and LlamaIndex service), and tests. The layout separates concerns between data preparation, database container, query service, and verification.

## Complexity Tracking

*Fill ONLY if Constitution Check has violations that must be justified*

No constitutional violations at this stage. The demo follows best practices for containerized applications with clear separation of concerns.
