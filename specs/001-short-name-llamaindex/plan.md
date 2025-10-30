# Implementation Plan: Llamaindex + RamaLama RAG Demo

**Branch**: `ambient-llamaindex-demo` | **Date**: 2025-10-30 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-short-name-llamaindex/spec.md`

## Execution Flow (/plan command scope)
```
1. Load feature spec from Input path
   → ✓ Feature spec loaded successfully
2. Fill Technical Context (scan for NEEDS CLARIFICATION)
   → Detecting project type and technical stack
   → Setting structure decision based on project analysis
3. Fill the Constitution Check section based on the content of the constitution document.
   → Constitution file is template-only, using minimal checks
4. Evaluate Constitution Check section below
   → If violations exist: Document in Complexity Tracking
   → If no justification possible: ERROR "Simplify approach first"
   → Update Progress Tracking: Initial Constitution Check
5. Execute Phase 0 → research.md
   → If NEEDS CLARIFICATION remain: ERROR "Resolve unknowns"
6. Execute Phase 1 → contracts, data-model.md, quickstart.md, CLAUDE.md
7. Re-evaluate Constitution Check section
   → If new violations: Refactor design, return to Phase 1
   → Update Progress Tracking: Post-Design Constitution Check
8. Plan Phase 2 → Describe task generation approach (DO NOT create tasks.md)
9. STOP - Ready for /tasks command
```

**IMPORTANT**: The /plan command STOPS at step 7. Phases 2-4 are executed by other commands:
- Phase 2: /tasks command creates tasks.md
- Phase 3-4: Implementation execution (manual or via tools)

## Summary

A containerized RAG demonstration system that integrates Llamaindex orchestration framework with RamaLama for local LLM serving. The system enables ML engineers and solution architects to deploy production-like RAG workflows in under 30 minutes, supporting document retrieval and generation with their own document collections. The demo provides a reference architecture for containerized RAG deployments with clear paths from local development to production.

## Technical Context

**Language/Version**: Python 3.12
**Primary Dependencies**: Llamaindex (RAG orchestration), RamaLama (LLM serving), FastAPI (API layer), PostgreSQL + pgvector (vector storage), existing docs2db-api infrastructure
**Storage**: PostgreSQL with pgvector extension for embeddings and document storage
**Testing**: pytest with async support, contract testing for API endpoints, integration tests for RAG pipeline
**Target Platform**: Linux containers (Podman/Docker), x86_64 CPU-based inference
**Project Type**: single (demo added to existing docs2db-api repository under /demos)
**Performance Goals**: Query responses <5s, system startup <2 minutes, vector search <1s for 1000 documents
**Constraints**: 4GB RAM limit, CPU-only inference, single embedding model consistency, no GPU required
**Scale/Scope**: Demo-scale deployment (100-1000 documents), single-user queries, local development focus

## Constitution Check
*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Note**: Constitution file is currently a template. Using minimal checks for this planning phase:

### Minimal Constitution Checks
- ✓ **Simplicity**: Demo scoped appropriately, uses existing infrastructure
- ✓ **Testability**: Clear functional requirements enable contract and integration tests
- ✓ **Documentation**: Quick-start and architecture docs explicitly required
- ? **Technology Choices**: Need to research Llamaindex integration patterns and RamaLama deployment
- ? **Dependencies**: Need to validate embedding model compatibility between components

**Initial Assessment**: PASS (with research required)

## Project Structure

### Documentation (this feature)
```
specs/001-short-name-llamaindex/
├── plan.md              # This file (/plan command output)
├── research.md          # Phase 0 output (/plan command)
├── data-model.md        # Phase 1 output (/plan command)
├── quickstart.md        # Phase 1 output (/plan command)
├── contracts/           # Phase 1 output (/plan command)
└── tasks.md             # Phase 2 output (/tasks command - NOT created by /plan)
```

### Source Code (docs2db-api repository)
```
demos/
└── llamaindex-ramalama/
    ├── README.md
    ├── compose.yml
    ├── src/
    │   ├── api/           # FastAPI application
    │   ├── rag/           # Llamaindex integration
    │   └── config/        # Configuration management
    ├── tests/
    │   ├── contract/      # API contract tests
    │   ├── integration/   # RAG pipeline tests
    │   └── unit/          # Component unit tests
    └── docs/
        ├── architecture.md
        └── troubleshooting.md
```

**Structure Decision**: Single project extension to existing docs2db-api repository. The demo will be self-contained under `/demos/llamaindex-ramalama/` with its own dependency management, allowing it to be deployed independently while leveraging the existing docs2db database infrastructure.

## Phase 0: Outline & Research

### Research Tasks Identified

From Technical Context unknowns:
1. **Llamaindex Integration Patterns**: How to integrate Llamaindex with existing pgvector database
2. **RamaLama Deployment**: Container deployment patterns for RamaLama model serving
3. **Embedding Compatibility**: Validation approaches for embedding model consistency
4. **Performance Optimization**: Vector search optimization for target scale

### Research Agent Tasks

1. **Task**: Research Llamaindex integration with PostgreSQL + pgvector
   - Focus: Connection patterns, index configuration, retrieval strategies
   - Output: Recommended approach for database integration

2. **Task**: Research RamaLama containerized deployment patterns
   - Focus: Model serving configuration, resource limits, health checks
   - Output: Container specification and startup sequence

3. **Task**: Research embedding model validation techniques
   - Focus: Dimension checking, model metadata storage, startup validation
   - Output: Validation approach for model consistency

4. **Task**: Research best practices for Llamaindex + local LLM serving
   - Focus: Prompt templating, context management, response streaming
   - Output: RAG pipeline configuration recommendations

**Output**: research.md with all technical decisions documented

## Phase 1: Design & Contracts
*Prerequisites: research.md complete*

### Design Deliverables

1. **data-model.md**: Entity definitions
   - Document, Chunk, Embedding (from existing schema)
   - Query, Response (new for demo)
   - Model Metadata (validation entity)
   - Relationships and validation rules

2. **contracts/**: API specifications
   - `POST /query` - Submit natural language query
   - `GET /health` - System health status
   - `GET /stats` - Database and model statistics
   - OpenAPI 3.0 schemas with request/response formats

3. **quickstart.md**: Step-by-step deployment guide
   - Prerequisites check
   - Database preparation
   - Container deployment
   - Example queries
   - Troubleshooting common issues

4. **CLAUDE.md**: Agent context file
   - Technology stack summary
   - Repository structure
   - Recent changes log
   - Development guidelines

### Contract Test Strategy

- Each endpoint gets dedicated contract test file
- Tests validate request/response schemas against OpenAPI spec
- Tests must fail initially (no implementation)
- Covers success cases and error responses

### Integration Test Scenarios

From user stories in spec.md:
- Scenario 1: End-to-end query processing with known documents
- Scenario 2: Health check reporting all component status
- Scenario 3: Embedding model mismatch detection
- Scenario 4: Empty database handling
- Scenario 5: System restart and recovery

**Output**: data-model.md, /contracts/*, failing contract tests, quickstart.md, CLAUDE.md

## Phase 2: Task Planning Approach
*This section describes what the /tasks command will do - DO NOT execute during /plan*

**Task Generation Strategy**:
- Load `.specify/templates/tasks-template.md` as base
- Generate tasks from Phase 1 design docs (contracts, data model, quickstart)
- Each contract → contract test task [P]
- Each RAG pipeline component → integration test + implementation
- Each configuration aspect → validation task
- Documentation tasks for architecture and troubleshooting

**Ordering Strategy**:
- TDD order: Tests before implementation
- Dependency order: Infrastructure → Models → Services → API
- Setup tasks first: Container configuration, database schema validation
- Core pipeline: Llamaindex integration → RamaLama integration → API layer
- Mark [P] for parallel execution (independent files)

**Estimated Output**: 20-25 numbered, ordered tasks in tasks.md

**IMPORTANT**: This phase is executed by the /tasks command, NOT by /plan

## Phase 3+: Future Implementation
*These phases are beyond the scope of the /plan command*

**Phase 3**: Task execution (/tasks command creates tasks.md)
**Phase 4**: Implementation (execute tasks.md following constitutional principles)
**Phase 5**: Validation (run tests, execute quickstart.md, performance validation)

## Complexity Tracking
*Fill ONLY if Constitution Check has violations that must be justified*

No violations requiring justification at this stage. Constitution file is template-only; full assessment will occur when constitution is defined.

## Progress Tracking
*This checklist is updated during execution flow*

**Phase Status**:
- [x] Phase 0: Research complete (/plan command) ✅ 2025-10-30
- [x] Phase 1: Design complete (/plan command) ✅ 2025-10-30
- [x] Phase 2: Task planning complete (/plan command - describe approach only) ✅ 2025-10-30
- [ ] Phase 3: Tasks generated (/tasks command)
- [ ] Phase 4: Implementation complete
- [ ] Phase 5: Validation passed

**Gate Status**:
- [x] Initial Constitution Check: PASS (minimal checks)
- [x] Post-Design Constitution Check: PASS (no design violations)
- [x] All NEEDS CLARIFICATION resolved
- [x] Complexity deviations documented (none)

**Artifacts Generated**:
- [x] research.md - Technology decisions and best practices
- [x] data-model.md - Entity definitions and database schema
- [x] contracts/openapi.yaml - REST API specification
- [x] quickstart.md - Deployment guide
- [x] CLAUDE.md - Agent context file

---
*Based on Constitution (template) - See `.specify/memory/constitution.md`*
