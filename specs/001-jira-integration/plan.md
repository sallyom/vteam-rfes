
# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]
**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

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
Enable automatic publishing of vTeam session artifacts (spec.md, tasks.md, plan.md) to Jira issues using the Atlassian MCP Server. When a workflow session completes, generated artifacts are automatically synchronized to the linked Jira issue, eliminating manual copy-paste and ensuring development teams have immediate access to planning artifacts in their existing workflow tools.

## Technical Context
**Language/Version**: Go 1.24+ (backend), TypeScript/Next.js 15+ (frontend)
**Primary Dependencies**: Gin web framework (backend), React 19 (frontend), Atlassian MCP Server client library
**Storage**: File system for artifacts, configuration store for OAuth credentials and session metadata
**Testing**: Go testing (backend), Jest/React Testing Library (frontend)
**Target Platform**: Linux server (backend), modern web browsers (frontend)
**Project Type**: web (backend + frontend)
**Performance Goals**: Artifact publishing <30s, support concurrent session completions, handle rate limits gracefully
**Constraints**: Atlassian Cloud only, OAuth 2.0 required, TLS 1.2+ encryption, respect Jira API rate limits, artifact size limits per Jira constraints
**Scale/Scope**: Support teams with 10-100 concurrent sessions, handle artifacts up to 10MB, integrate with existing vTeam session lifecycle

## Constitution Check
*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Library-First Architecture**:
- [x] Jira integration implemented as standalone library (components/backend/jira/)
- [x] OAuth client wrapper independently testable (oauth.go, client.go)
- [x] Publishing logic in separate, reusable modules (publisher.go, formatter.go)

**CLI/API Interface**:
- [x] REST API endpoints defined in jira-api.yaml for UI integration
- [x] JSON input/output for all operations (OpenAPI spec)
- [x] Backend library callable from API handlers and future CLI

**Test-First (NON-NEGOTIABLE)**:
- [x] Contract tests defined in jira-api.yaml (15 endpoints)
- [x] Integration tests planned in quickstart.md
- [x] OAuth flow tests with mock provider (in test plan)
- [x] Error handling tests for all status codes

**Integration Testing**:
- [x] End-to-end artifact publishing test (quickstart.md Step 6)
- [x] Retry mechanism test (quickstart.md Step 10)
- [x] Rate limit handling test (research.md Section 4)
- [x] Error scenarios covered (quickstart.md Step 9)

**Initial Assessment**: PASS - Feature aligns with library-first approach, testable interfaces, and integration requirements.

**Post-Design Assessment**: PASS - Design follows library-first architecture with clear separation of concerns. All components independently testable. REST API provides integration points. TDD approach with contract tests before implementation. No constitutional violations.

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

### Source Code (repository root)
```
components/backend/
├── jira/
│   ├── client.go           # MCP client wrapper
│   ├── publisher.go        # Artifact publishing logic
│   ├── oauth.go            # OAuth 2.0 authentication
│   ├── retry.go            # Retry mechanism with backoff
│   └── formatter.go        # Markdown to Jira format conversion
├── api/
│   └── jira_handlers.go    # REST endpoints for Jira integration
├── models/
│   └── jira_config.go      # Configuration and session metadata models
└── tests/
    ├── jira_test.go        # Unit tests
    └── integration/
        └── jira_integration_test.go

components/frontend/
├── src/
│   ├── components/
│   │   ├── JiraSettings.tsx      # Settings UI for OAuth config
│   │   └── JiraLinkDialog.tsx    # Link session to Jira issue
│   ├── services/
│   │   └── jiraService.ts        # Frontend API client
│   └── hooks/
│       └── useJiraIntegration.ts # React hook for Jira operations
└── tests/
    └── components/
        ├── JiraSettings.test.tsx
        └── JiraLinkDialog.test.tsx
```

**Structure Decision**: Web application structure with separate backend (Go) and frontend (Next.js/React) components. Jira integration follows existing vTeam architecture with library-first approach in backend and component-based UI in frontend.

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

## Phase 2: Task Planning Approach
*This section describes what the /tasks command will do - DO NOT execute during /plan*

**Task Generation Strategy**:

The /tasks command will load `.specify/templates/tasks-template.md` and generate tasks from Phase 1 artifacts:

1. **From data-model.md**:
   - Create Go structs for each entity (JiraConfiguration, SessionJiraLink, Artifact, PublishingOperation, JiraPublishQueue)
   - Define validation functions for each entity
   - Create database migration scripts (if using DB) or file storage setup

2. **From jira-api.yaml contract**:
   - Generate contract tests for each endpoint (15 endpoints)
   - Create API handler stubs in components/backend/api/jira_handlers.go
   - Generate frontend TypeScript types from OpenAPI schema

3. **From research.md decisions**:
   - Implement OAuth 2.0 authentication flow (authorize, callback, token refresh)
   - Integrate andygrunwald/go-jira or ctreminiom/go-atlassian library
   - Implement markdown-to-ADF conversion using summonio/markdown-to-adf
   - Create rate limiting with exponential backoff logic
   - Implement error classification and handling

4. **From quickstart.md scenarios**:
   - Integration test: End-to-end OAuth flow
   - Integration test: Link session to Jira issue
   - Integration test: Publish artifacts with attachments
   - Integration test: Publish with comments (markdown to ADF)
   - Integration test: Error handling (401, 403, 404, 429, 5xx)
   - Integration test: Retry mechanism

5. **Frontend components** (from contract endpoints):
   - JiraSettings.tsx component with OAuth authorization flow
   - JiraLinkDialog.tsx for linking sessions to issues
   - JiraPublishStatus.tsx for displaying publishing progress
   - jiraService.ts API client
   - useJiraIntegration.ts React hook

**Ordering Strategy**:

1. **Phase 1 - Data Layer** [P]:
   - Task: Define Go models (JiraConfiguration, SessionJiraLink, etc.)
   - Task: Create validation functions
   - Task: Set up storage (file system or database)

2. **Phase 2 - Core Libraries** [P]:
   - Task: Implement OAuth 2.0 client (authorize, callback, refresh)
   - Task: Integrate Jira REST API client library
   - Task: Implement markdown-to-ADF converter
   - Task: Create rate limiter with exponential backoff
   - Task: Implement error classifier

3. **Phase 3 - Publishing Logic**:
   - Task: Implement attachment upload handler
   - Task: Implement comment posting handler
   - Task: Implement publishing orchestrator (attachments + comments)
   - Task: Create retry queue and background worker

4. **Phase 4 - API Endpoints**:
   - Task: Implement /jira/config endpoints (GET, POST, DELETE)
   - Task: Implement /jira/oauth endpoints (authorize, callback)
   - Task: Implement /sessions/{id}/jira-link endpoints
   - Task: Implement /sessions/{id}/publish endpoints
   - Task: Implement /jira/health and /jira/operations endpoints

5. **Phase 5 - Frontend Components** [P]:
   - Task: Create JiraSettings component with OAuth flow
   - Task: Create JiraLinkDialog component
   - Task: Create JiraPublishStatus component
   - Task: Implement jiraService API client
   - Task: Create useJiraIntegration React hook

6. **Phase 6 - Integration Tests**:
   - Task: OAuth flow end-to-end test
   - Task: Artifact publishing test with mock Jira
   - Task: Error handling tests (all error codes)
   - Task: Retry mechanism test
   - Task: Rate limiting test

7. **Phase 7 - Documentation & Deployment**:
   - Task: Write OAuth setup guide with screenshots
   - Task: Write user guide for linking sessions
   - Task: Create troubleshooting documentation
   - Task: Set up monitoring/alerting for publishing operations
   - Task: Configure secrets management for OAuth credentials

**Dependency Relationships**:
- Phase 1 must complete before Phase 2-3
- Phase 2 must complete before Phase 3-4
- Phase 3 must complete before Phase 4
- Phase 4 backend can run parallel with Phase 5 frontend
- Phase 6 depends on Phase 4-5 completion
- Phase 7 can start after Phase 4 is stable

**Parallel Execution** [P]:
- All tasks within Phase 1 can run parallel (independent models)
- All tasks within Phase 2 can run parallel (independent libraries)
- All tasks within Phase 5 can run parallel (independent React components)
- Contract test creation can run parallel with implementation

**Estimated Output**: 40-45 numbered, dependency-ordered tasks in tasks.md

**Task Breakdown Philosophy**:
- Each task: 2-4 hours of focused work
- Tasks are independently testable
- Clear acceptance criteria per task
- TDD approach: Write tests, see them fail, then implement

**IMPORTANT**: This phase is executed by the /tasks command, NOT by /plan

## Phase 3+: Future Implementation
*These phases are beyond the scope of the /plan command*

**Phase 3**: Task execution (/tasks command creates tasks.md)  
**Phase 4**: Implementation (execute tasks.md following constitutional principles)  
**Phase 5**: Validation (run tests, execute quickstart.md, performance validation)

## Complexity Tracking
*Fill ONLY if Constitution Check has violations that must be justified*

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |


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
- [x] Initial Constitution Check: PASS
- [x] Post-Design Constitution Check: PASS
- [x] All NEEDS CLARIFICATION resolved (from research phase)
- [x] Complexity deviations documented (none - no constitutional violations)

---
*Based on Constitution v2.1.1 - See `/memory/constitution.md`*
