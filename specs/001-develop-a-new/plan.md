
# Implementation Plan: Jira-Session Artifact Integration

**Branch**: `001-develop-a-new` | **Date**: 2025-10-09 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/workspace/sessions/agentic-session-1760033087/workspace/vteam-rfes/specs/001-develop-a-new/spec.md`

## Execution Flow (/plan command scope)
```
1. Load feature spec from Input path
   → If not found: ERROR "No feature spec at {path}"
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
6. Execute Phase 1 → contracts, data-model.md, quickstart.md, agent-specific template file (e.g., `CLAUDE.md` for Claude Code, `.github/copilot-instructions.md` for GitHub Copilot, `GEMINI.md` for Gemini CLI, `QWEN.md` for Qwen Code or `AGENTS.md` for opencode).
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
This feature enables vTeam users to link agentic sessions to Jira issues and push session artifacts (transcripts, results, state files) as attachments and comments. Users can manually trigger push operations via a UI dialog after session completion, with project-scoped Jira authentication via runner secrets. The system validates Jira connectivity, handles errors gracefully, and maintains push history in session metadata for audit trails.

## Technical Context
**Language/Version**: Go 1.21+ (backend), TypeScript 5.x (frontend - Next.js)
**Primary Dependencies**: Go net/http, Kubernetes client-go, Next.js, React, Jira REST API client
**Storage**: AgenticSession Custom Resource (Kubernetes), session state files (PVC/stateDir), Jira cloud/server
**Testing**: Go testing package, React Testing Library, contract tests for Jira API
**Target Platform**: Kubernetes cluster (backend pods), web browser (frontend)
**Project Type**: web (components/backend + components/frontend)
**Performance Goals**: <3s for artifact upload, <500ms for Jira validation, handle 10MB+ artifacts
**Constraints**: Jira API rate limits (300-1000 req/min), 10MB attachment size limit per file, network latency for external Jira calls
**Scale/Scope**: ~100 concurrent sessions per cluster, ~50 artifacts per session, support 10+ projects with separate Jira configs

## Constitution Check
*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Status**: PASS (no specific constitution defined - using general best practices)

Since the constitution.md is a template without specific principles, this feature will follow general software engineering best practices:
- Testable components with clear contracts
- Separation of concerns (UI, API, Kubernetes integration)
- Error handling and observability
- No new architectural complexity - extends existing patterns in vTeam

## Project Structure

### Documentation (this feature)
```
specs/[###-feature]/
├── plan.md              # This file (/plan command output)
├── research.md          # Phase 0 output (/plan command)
├── data-model.md        # Phase 1 output (/plan command)
├── quickstart.md        # Phase 1 output (/plan command)
├── contracts/           # Phase 1 output (/plan command)
└── tasks.md             # Phase 2 output (/tasks command - NOT created by /plan)
```

### Source Code (repository root: /workspace/sessions/agentic-session-1760033087/workspace/vTeam)
```
components/backend/
├── handlers.go           # HTTP API handlers - will add Jira push endpoints
├── main.go              # Server initialization and routing
├── github_app.go        # Existing GitHub integration - pattern reference
└── (tests to be added)

components/frontend/
├── src/
│   ├── app/             # Next.js app router pages
│   ├── components/      # React components - will add Jira push dialog
│   ├── lib/             # API client utilities
│   └── types/           # TypeScript type definitions
└── (tests to be added)

components/manifests/
└── operator/
    └── agenticsession-crd.yaml  # Custom Resource Definition - may need annotation updates

components/runners/
└── (session execution context - artifact source)
```

**Structure Decision**: Web application architecture with Go backend and Next.js frontend. This feature extends existing patterns for GitHub integration (backend/github_app.go) and session management. New code will follow the existing handler-based API pattern in backend and component-based UI in frontend.

## Phase 0: Outline & Research
1. **Extract unknowns from Technical Context** above:
   - For each NEEDS CLARIFICATION → research task
   - For each dependency → best practices task
   - For each integration → patterns task

2. **Generate and dispatch research agents**:
   ```
   For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"
   ```

3. **Consolidate findings** in `research.md` using format:
   - Decision: [what was chosen]
   - Rationale: [why chosen]
   - Alternatives considered: [what else evaluated]

**Output**: research.md with all NEEDS CLARIFICATION resolved

## Phase 1: Design & Contracts
*Prerequisites: research.md complete*

1. **Extract entities from feature spec** → `data-model.md`:
   - Entity name, fields, relationships
   - Validation rules from requirements
   - State transitions if applicable

2. **Generate API contracts** from functional requirements:
   - For each user action → endpoint
   - Use standard REST/GraphQL patterns
   - Output OpenAPI/GraphQL schema to `/contracts/`

3. **Generate contract tests** from contracts:
   - One test file per endpoint
   - Assert request/response schemas
   - Tests must fail (no implementation yet)

4. **Extract test scenarios** from user stories:
   - Each story → integration test scenario
   - Quickstart test = story validation steps

5. **Update agent file incrementally** (O(1) operation):
   - Run `.specify/scripts/bash/update-agent-context.sh cursor`
     **IMPORTANT**: Execute it exactly as specified above. Do not add or remove any arguments.
   - If exists: Add only NEW tech from current plan
   - Preserve manual additions between markers
   - Update recent changes (keep last 3)
   - Keep under 150 lines for token efficiency
   - Output to repository root

**Output**: data-model.md, /contracts/*, failing tests, quickstart.md, agent-specific file

**Phase 1 Artifacts Created**:
- ✅ data-model.md: Complete data structures and relationships
- ✅ contracts/session-jira-api.yaml: OpenAPI 3.0 specification for 4 endpoints
- ✅ quickstart.md: 11 acceptance test scenarios with validation steps
- ⚠️ Contract tests: TO BE CREATED during implementation (require Go test infrastructure)
- ⚠️ Agent context update: TO BE RUN in vTeam repo (not vteam-rfes)

## Phase 2: Task Planning Approach
*This section describes what the /tasks command will do - DO NOT execute during /plan*

**Task Generation Strategy**:
1. Load `.specify/templates/tasks-template.md` as base
2. Generate backend tasks from contracts and data model:
   - Backend handler functions (listSessionArtifacts, validateSessionJiraIssue, pushSessionToJira, getSessionJiraLinks)
   - Jira API client wrapper (based on existing RFEWorkflow pattern)
   - Annotation marshaling helpers (JiraLink serialization)
   - Contract tests for each endpoint (4 test files)
3. Generate frontend tasks from UI/UX patterns:
   - PushToJiraDialog component
   - JiraLinkBadge component
   - Push history view component
   - API client functions
4. Generate integration tests from quickstart scenarios:
   - End-to-end push workflow test
   - Error handling tests (missing config, invalid token, network failure)
5. Generate documentation tasks:
   - User guide for Jira configuration
   - API documentation
   - Update vTeam CLAUDE.md with new context

**Ordering Strategy**:
- **Foundation** (parallel): Data model types, API contracts
- **Backend Core**: Handler functions, Jira client, annotation helpers
- **Backend Tests**: Contract tests (can run in parallel), integration tests
- **Frontend Core**: Dialog component, badge component, API client
- **Frontend Tests**: Component tests, E2E tests
- **Integration**: Connect frontend to backend, validate full workflow
- **Documentation**: Update guides and context files

**Estimated Task Breakdown**:
- Backend: 12-15 tasks (handlers, client, helpers, tests)
- Frontend: 10-12 tasks (components, API, tests)
- Integration: 3-5 tasks (E2E tests, validation)
- Documentation: 3-4 tasks (guides, context)
- **Total**: 28-36 numbered, dependency-ordered tasks

**Parallel Execution Opportunities** (mark [P]):
- Backend handler functions (independent files)
- Frontend components (independent files)
- Contract test files (independent test suites)
- Documentation updates (independent docs)

**Key Dependencies**:
```
Data Model → Backend Handlers → Contract Tests
          → Frontend Types → UI Components
Backend Handlers + UI Components → Integration Tests
All Implementation → Documentation
```

**IMPORTANT**: This phase is executed by the /tasks command, NOT by /plan

## Phase 3+: Future Implementation
*These phases are beyond the scope of the /plan command*

**Phase 3**: Task execution (/tasks command creates tasks.md)  
**Phase 4**: Implementation (execute tasks.md following constitutional principles)  
**Phase 5**: Validation (run tests, execute quickstart.md, performance validation)

## Complexity Tracking
*Fill ONLY if Constitution Check has violations that must be justified*

**Status**: No constitutional violations

This feature extends existing patterns in vTeam without introducing new architectural complexity:
- Reuses RFEWorkflow Jira integration patterns (handlers.go:2879-3141)
- No new services or infrastructure components
- Uses existing Kubernetes primitives (annotations, secrets)
- Follows established API handler structure
- Maintains multi-tenant isolation via project-scoped secrets

No deviations from best practices to document.


## Progress Tracking
*This checklist is updated during execution flow*

**Phase Status**:
- [x] Phase 0: Research complete (/plan command) - research.md created
- [x] Phase 1: Design complete (/plan command) - data-model.md, contracts/, quickstart.md created
- [x] Phase 2: Task planning complete (/plan command - approach documented above)
- [ ] Phase 3: Tasks generated (/tasks command) - NOT executed by /plan
- [ ] Phase 4: Implementation complete
- [ ] Phase 5: Validation passed

**Gate Status**:
- [x] Initial Constitution Check: PASS
- [x] Post-Design Constitution Check: PASS
- [x] All NEEDS CLARIFICATION resolved (Technical Context fully populated)
- [x] Complexity deviations documented (None - extends existing patterns)

**Artifacts Generated**:
- [x] research.md (11 decisions with rationale)
- [x] data-model.md (9 sections: entities, schemas, validations)
- [x] contracts/session-jira-api.yaml (OpenAPI 3.0 spec, 4 endpoints, 13 schemas)
- [x] quickstart.md (11 test scenarios with acceptance criteria)
- [ ] tasks.md (to be created by /tasks command)

---
*Based on Constitution v2.1.1 - See `/memory/constitution.md`*
