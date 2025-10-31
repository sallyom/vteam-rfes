
# Implementation Plan: BugFix Workspace Type

**Branch**: `001-bugfix-workspace` | **Date**: 2025-10-31 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-bugfix-workspace/spec.md`

## Execution Flow (/plan command scope)
```
1. Load feature spec from Input path
   → SUCCESS: Spec loaded from specs/001-bugfix-workspace/spec.md
2. Fill Technical Context (scan for NEEDS CLARIFICATION)
   → Detect Project Type from file system structure or context (web=frontend+backend, mobile=app+api)
   → Set Structure Decision based on project type
3. Fill the Constitution Check section based on the content of the constitution document.
4. Evaluate Constitution Check section below
   → If violations exist: Document in Complexity Tracking
   → If no justification possible: ERROR "Simplify approach first"
   → Update Progress Tracking: Initial Constitution Check
5. Execute Phase 0 → research.md
   → If NEEDS CLARIFICATION remain: ERROR "Resolve unknowns"
6. Execute Phase 1 → contracts, data-model.md, quickstart.md, agent-specific template file
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
The BugFix Workspace Type provides developers with a structured workflow for bug identification, resolution planning, and implementation tracking through specialized sessions (Bug-review, Bug-resolution-plan, Bug-implement-fix, and generic). The feature integrates with GitHub Issues as the primary tracking artifact, with optional Jira synchronization for project management visibility. Implementation will extend the existing vTeam platform (Go backend + Next.js frontend) by adding a new Workspace type alongside existing RFE workspaces, utilizing the established Kubernetes operator pattern for workspace orchestration.

## Technical Context
**Language/Version**: Go 1.24.0 (backend), TypeScript/Next.js 15+ (frontend)
**Primary Dependencies**: Gin (Go web framework), Kubernetes client-go, Next.js, React 19, Radix UI, WebSocket (gorilla/websocket)
**Storage**: Kubernetes Custom Resources (CRDs) for workspace state, Git repositories for documentation artifacts
**Testing**: Go testing package (backend), Jest/React Testing Library (frontend)
**Target Platform**: Kubernetes clusters (OpenShift compatible), Web browsers (frontend)
**Project Type**: web - frontend + backend architecture
**Performance Goals**: Workspace creation <5s, GitHub API calls <2s, Jira sync <10s, WebSocket real-time updates <100ms
**Constraints**: Kubernetes API compatibility, GitHub API rate limits (5000/hour authenticated), Jira API rate limits (varies by plan), offline operation not supported
**Scale/Scope**: 100+ concurrent workspaces, 1000+ bugs tracked per developer annually, 50+ simultaneous WebSocket connections

## Constitution Check
*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Note**: The constitution file in `.specify/memory/constitution.md` is a template and has not been customized for this project. Since no project-specific constitution exists, we will apply general software engineering principles and verify against the feature specification requirements.

**General Principles Check**:
- **Modularity**: BugFix Workspace should be implemented as a distinct module/component
- **Testing**: All features must have comprehensive test coverage
- **Documentation**: Clear developer and user documentation required
- **API Design**: RESTful patterns for backend, component-based architecture for frontend
- **Error Handling**: Graceful degradation for GitHub/Jira API failures

**Violations**: None identified at planning stage. The feature follows established patterns in the vTeam codebase.

## Project Structure

### Documentation (this feature)
```
specs/001-bugfix-workspace/
├── plan.md              # This file (/plan command output)
├── research.md          # Phase 0 output (/plan command)
├── data-model.md        # Phase 1 output (/plan command)
├── quickstart.md        # Phase 1 output (/plan command)
├── contracts/           # Phase 1 output (/plan command)
└── tasks.md             # Phase 2 output (/tasks command - NOT created by /plan)
```

### Source Code (vTeam repository: /workspace/sessions/agentic-session-1761878941/workspace/vTeam)
```
components/
├── backend/             # Go backend services
│   ├── handlers/        # HTTP request handlers
│   │   └── bugfix/      # NEW: BugFix workspace handlers
│   ├── types/           # Data type definitions
│   │   └── bugfix.go    # NEW: BugFix workspace types
│   ├── github/          # GitHub API integration (existing)
│   ├── jira/            # Jira API integration (existing)
│   ├── k8s/             # Kubernetes client operations
│   │   └── bugfix/      # NEW: BugFix CRD operations
│   ├── crd/             # Custom Resource Definitions
│   │   └── bugfixworkspace_v1alpha1.yaml  # NEW: BugFix CRD
│   └── websocket/       # WebSocket handlers (existing)
│
├── frontend/            # Next.js frontend
│   └── src/
│       ├── app/
│       │   └── workspaces/
│       │       └── bugfix/   # NEW: BugFix workspace pages
│       │           ├── page.tsx
│       │           ├── [id]/
│       │           │   └── page.tsx
│       │           └── sessions/
│       │               ├── bug-review.tsx
│       │               ├── bug-resolution-plan.tsx
│       │               ├── bug-implement-fix.tsx
│       │               └── generic.tsx
│       ├── components/
│       │   └── workspaces/
│       │       └── bugfix/   # NEW: BugFix UI components
│       │           ├── WorkspaceCreator.tsx
│       │           ├── SessionSelector.tsx
│       │           ├── JiraSyncButton.tsx
│       │           └── BugTimeline.tsx
│       └── lib/
│           └── api/
│               └── bugfix.ts  # NEW: BugFix API client

tests/
├── backend/
│   └── bugfix/          # NEW: Backend tests
│       ├── handlers_test.go
│       ├── types_test.go
│       └── integration/
└── frontend/
    └── bugfix/          # NEW: Frontend tests
        ├── WorkspaceCreator.test.tsx
        └── SessionSelector.test.tsx
```

**Structure Decision**: Web application architecture. The vTeam platform uses a microservices-style component organization with separate backend (Go) and frontend (Next.js) directories. The BugFix Workspace will follow the established pattern of workspace types, integrating with existing GitHub/Jira client code and Kubernetes operator infrastructure. New code will be added under `handlers/bugfix/`, `types/bugfix.go`, `k8s/bugfix/` (backend) and `app/workspaces/bugfix/`, `components/workspaces/bugfix/` (frontend).

## Phase 0: Outline & Research
1. **Extract unknowns from Technical Context** above:
   - GitHub API patterns for issue creation/updates and comment management
   - Jira API patterns for task creation/updates and bidirectional linking
   - Kubernetes CRD design for BugFix Workspace state management
   - WebSocket event patterns for real-time session updates
   - Spec Repository Git operations (clone, commit, push) from backend
   - Session orchestration patterns (agent invocation, state persistence)
   - Authentication/authorization for GitHub and Jira (token management)

2. **Research tasks to dispatch**:
   - GitHub API v3/v4: Issue CRUD, comments, labels, linking (current implementation in `components/backend/github/`)
   - Jira REST API: Issue creation, comments, bidirectional linking (current implementation in `components/backend/jira/`)
   - Kubernetes CRD best practices: Workspace lifecycle, status conditions, finalizers
   - vTeam workspace patterns: How existing RFE workspaces are structured and managed
   - Git operations from Go: go-git library patterns, authentication, push/pull
   - Agent integration: How vTeam currently invokes/manages Claude agents for sessions
   - Session state management: Persistence patterns for multi-step workflows

3. **Consolidate findings** in `research.md` using format:
   - Decision: [what was chosen]
   - Rationale: [why chosen]
   - Alternatives considered: [what else evaluated]

**Output**: research.md with all unknowns resolved

## Phase 1: Design & Contracts
*Prerequisites: research.md complete*

1. **Extract entities from feature spec** → `data-model.md`:
   - BugFix Workspace (state, metadata, session history)
   - Bug Documentation artifact (bugfix-gh-<issue-number>.md structure)
   - GitHub Issue (external entity, sync state)
   - Jira Task (external entity, optional, sync state)
   - Session (Bug-review, Bug-resolution-plan, Bug-implement-fix, generic)
   - Feature Branch (branch metadata, PR links)
   - Spec Repository Bug Folder (folder structure, artifact management)

2. **Generate API contracts** from functional requirements:
   - POST   /api/v1/workspaces/bugfix              - Create BugFix Workspace (FR-001, FR-002, FR-003)
   - GET    /api/v1/workspaces/bugfix/:id          - Get workspace details
   - GET    /api/v1/workspaces/bugfix              - List all workspaces
   - DELETE /api/v1/workspaces/bugfix/:id          - Delete workspace
   - POST   /api/v1/workspaces/bugfix/:id/sessions - Start a session (FR-007)
   - POST   /api/v1/workspaces/bugfix/:id/sync-jira - Sync to Jira (FR-012, FR-027)
   - GET    /api/v1/workspaces/bugfix/:id/status   - Get workspace and session status
   - WebSocket /ws/workspaces/bugfix/:id            - Real-time session updates
   - Output: OpenAPI 3.0 spec to `/contracts/bugfix-api.yaml`

3. **Generate contract tests** from contracts:
   - Test files for each endpoint (handlers_test.go)
   - Assert request/response schemas match OpenAPI spec
   - Tests written but implementation not yet complete (TDD)

4. **Extract test scenarios** from user stories:
   - Scenario 1: Create workspace from GitHub Issue URL → integration test
   - Scenario 2: Create workspace from text description → integration test
   - Scenario 3: Bug-review session analysis → integration test
   - Scenario 4-8: Jira sync workflows → integration tests
   - Scenario 9: Generic session → integration test
   - Scenario 10: Developer portfolio access → integration test
   - Output: Integration test scenarios in quickstart.md

5. **Update agent file incrementally** (O(1) operation):
   - Run `.specify/scripts/bash/update-agent-context.sh claude`
   - Add BugFix Workspace context, session types, GitHub/Jira integration patterns
   - Update recent changes (keep last 3)
   - Keep under 150 lines for token efficiency

**Output**: data-model.md, /contracts/*, failing contract tests, quickstart.md, CLAUDE.md update

## Phase 2: Task Planning Approach
*This section describes what the /tasks command will do - DO NOT execute during /plan*

**Task Generation Strategy**:
- Load `.specify/templates/tasks-template.md` as base
- Generate tasks from Phase 1 design docs (contracts, data model, quickstart)
- Backend tasks:
  - Define BugFix CRD YAML (Kubernetes resource)
  - Implement BugFix types and state management
  - Create GitHub issue CRUD handlers (FR-002, FR-003)
  - Create Jira sync handlers (FR-013-FR-016, FR-028-FR-029)
  - Implement session orchestration (FR-006-FR-008)
  - Implement Bug-review session logic (FR-009-FR-012)
  - Implement Bug-resolution-plan session logic (FR-017-FR-020)
  - Implement Bug-implement-fix session logic (FR-021-FR-027)
  - Implement generic session logic (FR-030-FR-031)
  - Add Spec Repository Git operations (FR-004, FR-018, FR-025)
  - Implement WebSocket event broadcasting
  - Add error handling and rate limit management (NFR-002, NFR-003, NFR-006)
- Frontend tasks:
  - Create BugFix workspace creation UI (FR-001)
  - Build session selector component (FR-007)
  - Implement Jira sync button component (FR-012, FR-027)
  - Build session status/timeline view
  - Add WebSocket connection for real-time updates
  - Create bug documentation viewer
  - Implement workspace list and detail pages
- Integration tasks:
  - Contract tests for all endpoints
  - Integration tests for user scenarios 1-10
  - End-to-end test: Full bug workflow (create → review → plan → implement → sync)

**Ordering Strategy**:
- TDD order: Contract tests → Implementation → Integration tests
- Backend dependency order: CRD → Types → Handlers → Sessions → Integration
- Frontend dependency order: API client → Components → Pages → Integration
- Mark [P] for parallel execution:
  - Frontend and backend can be developed in parallel after contracts
  - Individual session type implementations are parallel
  - UI components are parallel

**Estimated Output**: 45-50 numbered, ordered tasks in tasks.md

**IMPORTANT**: This phase is executed by the /tasks command, NOT by /plan

## Phase 3+: Future Implementation
*These phases are beyond the scope of the /plan command*

**Phase 3**: Task execution (/tasks command creates tasks.md)
**Phase 4**: Implementation (execute tasks.md following constitutional principles)
**Phase 5**: Validation (run tests, execute quickstart.md, performance validation per Success Criteria SC-001 through SC-012)

## Complexity Tracking
*Fill ONLY if Constitution Check has violations that must be justified*

No constitution violations identified. The feature follows established architectural patterns in the vTeam codebase and does not introduce unnecessary complexity.

## Progress Tracking
*This checklist is updated during execution flow*

**Phase Status**:
- [x] Phase 0: Research complete (/plan command)
- [x] Phase 1: Design complete (/plan command)
- [x] Phase 2: Task planning complete (/plan command - describe approach only)
- [ ] Phase 3: Tasks generated (/tasks command)
- [ ] Phase 4: Implementation complete
- [ ] Phase 5: Validation passed

**Gate Status**:
- [x] Initial Constitution Check: PASS (no project-specific constitution, general principles applied)
- [x] Post-Design Constitution Check: PASS (no violations introduced during design)
- [x] All NEEDS CLARIFICATION resolved (all research complete)
- [x] Complexity deviations documented (none)

---
*Based on vTeam architecture - See vTeam repository and existing workspace patterns*
