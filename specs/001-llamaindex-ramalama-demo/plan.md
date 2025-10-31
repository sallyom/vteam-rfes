# Implementation Plan: Llamaindex RAG Demo with Ramalama

**Branch**: `001-llamaindex-ramalama-demo` | **Date**: 2025-10-31 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-llamaindex-ramalama-demo/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Create a containerized RAG (Retrieval-Augmented Generation) demonstration system that integrates Llamaindex framework with Ramalama model serving and a PGVector database pre-populated with docs2db-generated embeddings. The system will provide both programmatic and web-based interfaces for querying documentation, similar to the existing llamastack demo architecture.

## Technical Context

**Language/Version**: Python 3.11+ (required by Llamaindex and Ramalama)
**Primary Dependencies**: Llamaindex (RAG framework), Ramalama (model serving), PGVector (vector database), PostgreSQL with pgvector extension
**Storage**: PostgreSQL with PGVector extension for vector embeddings, pre-populated with docs2db-generated database dump
**Testing**: pytest for Python components, container health checks for deployment validation
**Target Platform**: Linux containers (Podman/Docker), tested on Linux/macOS/WSL2
**Project Type**: Web application (backend API + frontend UI + containerized services)
**Performance Goals**: Query responses under 5 seconds for typical documentation queries, support 10 concurrent queries without degradation
**Constraints**: 15-minute deployment time (excluding model downloads), minimum 8GB RAM and 20GB disk on host system, <200ms p95 for simple vector lookups
**Scale/Scope**: Demo/proof-of-concept system, single-user or small team usage, pre-populated database (no ingestion pipeline), similar architecture to existing llamastack demo

**Clarifications Needed**:
- NEEDS CLARIFICATION: Specific Ramalama model serving API and configuration requirements
- NEEDS CLARIFICATION: Llamaindex-PGVector integration patterns and best practices
- NEEDS CLARIFICATION: Container orchestration approach (pod configuration vs docker-compose)
- NEEDS CLARIFICATION: Web UI framework/technology selection
- NEEDS CLARIFICATION: Llamastack demo architecture details for reference
- NEEDS CLARIFICATION: docs2db database dump format and import mechanism

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Status**: Constitution file is currently a template at `.specify/memory/constitution.md`. No project-specific principles have been defined yet. Proceeding with standard best practices:

- ✓ **Modularity**: Design will separate container services (database, model server, RAG service, web UI)
- ✓ **Testability**: Each component will have health checks and validation scripts
- ✓ **Documentation**: Comprehensive setup and usage documentation required (FR-007)
- ✓ **Observability**: Logging and health endpoints required (FR-010, FR-011)
- ⚠️ **Demo Scope**: This is a demonstration system, not production-ready (explicitly out of scope: security hardening, auth, HA)

**No violations to justify at this stage.** Re-evaluation will occur after Phase 1 design.

---

### Post-Design Re-Evaluation (Phase 1 Complete)

After completing Phase 0 research and Phase 1 design artifacts (data-model.md, contracts/, quickstart.md), the design has been validated against best practices:

**✅ Design Validation Results**:

1. **Modularity**: ✓ PASS
   - 4 separate containerized services (database, model server, RAG service, UI)
   - Each service has well-defined responsibilities and interfaces
   - Services communicate via standard protocols (HTTP REST, PostgreSQL wire protocol)
   - Clear separation of concerns: storage, computation, orchestration, presentation

2. **Testability**: ✓ PASS
   - Health check endpoints on all services (/health, /ready)
   - Integration test hooks (retrieve endpoint for testing retrieval without LLM)
   - Container health probes (readiness, liveness, startup)
   - Example test queries and scripts provided in quickstart.md
   - Database statistics endpoint for monitoring

3. **Documentation**: ✓ PASS
   - Comprehensive quickstart.md with step-by-step deployment
   - OpenAPI 3.0 specification for REST API (rag-service-api.yaml)
   - Data model documentation with schema definitions
   - Research documentation with architectural decisions
   - Inline examples and troubleshooting guides

4. **Observability**: ✓ PASS
   - Health and readiness endpoints with dependency checks
   - Latency breakdowns in query responses (embedding, retrieval, generation)
   - Database statistics endpoint
   - Container logs accessible via podman/docker logs
   - Request/response metadata tracking (query_id, timestamps)

5. **Demo Scope**: ✓ APPROPRIATE
   - Clear documentation of out-of-scope items (auth, HA, security hardening)
   - Resource requirements documented (8GB RAM minimum)
   - Performance expectations set (5s query latency, 10 concurrent queries)
   - Deployment time target: 15-30 minutes
   - Suitable for demonstration and proof-of-concept purposes

**Architectural Decisions Validated**:
- ✓ Podman pod orchestration matches reference demo (llamastack)
- ✓ Shared localhost networking simplifies configuration
- ✓ Persistent volumes for database and models prevent data loss
- ✓ HNSW indexing provides <200ms p95 vector search latency
- ✓ Hybrid search (vector + text) maximizes retrieval quality
- ✓ OpenAI-compatible endpoints enable easy LLM swapping

**No constitution violations identified.** Design aligns with best practices for a demonstration system.

## Project Structure

### Documentation (this feature)

```
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```
llamaindex-ramalama-demo/
├── containers/
│   ├── database/
│   │   ├── Containerfile
│   │   ├── init-scripts/
│   │   └── import-dump.sh
│   ├── ramalama/
│   │   ├── Containerfile
│   │   └── config/
│   ├── rag-service/
│   │   ├── Containerfile
│   │   ├── src/
│   │   │   ├── main.py
│   │   │   ├── query_handler.py
│   │   │   └── config.py
│   │   └── requirements.txt
│   └── web-ui/
│       ├── Containerfile
│       ├── src/
│       └── public/
├── pod/
│   └── llamaindex-demo-pod.yaml
├── scripts/
│   ├── build-containers.sh
│   ├── deploy-pod.sh
│   ├── run-query.py
│   └── health-check.sh
├── tests/
│   ├── integration/
│   │   ├── test_deployment.py
│   │   ├── test_queries.py
│   │   └── test_health_checks.py
│   └── unit/
│       └── test_query_handler.py
├── examples/
│   └── sample-queries.json
└── docs/
    ├── SETUP.md
    ├── DEPLOYMENT.md
    └── TROUBLESHOOTING.md
```

**Structure Decision**: Web application with containerized services structure. The demo is organized around container definitions for each service component (database, model server, RAG service, web UI), with pod configuration for orchestration. This matches the existing llamastack demo pattern referenced in requirements and supports the containerized deployment requirement (FR-001).

## Complexity Tracking

*Fill ONLY if Constitution Check has violations that must be justified*

**No violations to track.** The design follows a straightforward containerized microservices pattern appropriate for a demonstration system.

