# Tasks: Jira-Session Artifact Integration

**Input**: Design documents from `/workspace/sessions/agentic-session-1760033971/workspace/vteam-rfes/specs/001-develop-a-new/`
**Prerequisites**: plan.md, research.md, data-model.md, contracts/session-jira-api.yaml, quickstart.md
**Repository Root**: `/workspace/sessions/agentic-session-1760033087/workspace/vTeam`

## Execution Flow
```
1. Load plan.md → Extract tech stack (Go 1.21+, TypeScript 5.x, Next.js)
2. Load design documents:
   → data-model.md: 9 sections (JiraLink, SessionArtifact, etc.)
   → contracts/session-jira-api.yaml: 4 endpoints
   → quickstart.md: 11 test scenarios
   → research.md: 11 decisions
3. Generate tasks by category:
   → Setup: Go types, API routes
   → Tests: Contract tests for 4 endpoints
   → Core: Backend handlers, Jira client, frontend components
   → Integration: E2E tests, validation
   → Documentation: User guide, context updates
4. Apply task rules:
   → Different files = [P] for parallel
   → Backend handlers in handlers.go = sequential
   → Tests before implementation (TDD)
5. Numbered tasks: T001-T036
6. Dependencies: Tests → Implementation → Integration → Documentation
```

## Format: `[ID] [P?] Description`
- **[P]**: Can run in parallel (different files, no dependencies)
- File paths are relative to `/workspace/sessions/agentic-session-1760033087/workspace/vTeam`

---

## Phase 3.1: Setup

- [X] **T001** Create Go types for Jira integration in `components/backend/jira_types.go`
  - Types: JiraLink, SessionArtifact, JiraConfiguration, PushRequest, PushResponse, ValidateIssueRequest, ValidateIssueResponse, JiraLinksResponse, ErrorResponse
  - Validation functions: validateIssueKey, validateArtifactSize
  - JSON struct tags for API serialization

- [X] **T002** [P] Create TypeScript types for Jira integration in `components/frontend/src/types/jira.ts`
  - Interfaces: JiraLink, SessionArtifact, PushRequest, PushResponse, ValidateIssueRequest, ValidateIssueResponse, ErrorResponse
  - Match OpenAPI schema from contracts/session-jira-api.yaml

- [X] **T003** Add API routes for Jira endpoints in `components/backend/main.go`
  - Register routes: GET /sessions/:sessionName/artifacts, POST /sessions/:sessionName/jira/validate, POST /sessions/:sessionName/jira, GET /sessions/:sessionName/jira
  - Map to handler functions (implemented in jira_handlers.go)

---

## Phase 3.2: Tests First (TDD) ⚠️ MUST COMPLETE BEFORE 3.3

**CRITICAL: These tests MUST be written and MUST FAIL before ANY implementation**

- [X] **T004** [P] Contract test for GET /artifacts endpoint in `components/backend/handlers_test.go`
  - Test: listSessionArtifacts returns 200 with artifact array
  - Test: Returns 404 for non-existent session
  - Test: Returns 401 for invalid user token
  - Mock stateDir with test artifacts
  - Verify response matches ArtifactListResponse schema

- [X] **T005** [P] Contract test for POST /jira/validate endpoint in `components/backend/handlers_test.go`
  - Test: validateSessionJiraIssue returns 200 with valid=true for valid issue
  - Test: Returns 200 with valid=false for invalid issue key
  - Test: Returns 400 for malformed request
  - Mock Jira API responses
  - Verify response matches ValidateIssueResponse schema

- [X] **T006** [P] Contract test for POST /jira endpoint in `components/backend/handlers_test.go`
  - Test: pushSessionToJira returns 200 with success=true
  - Test: Handles partial failures (some artifacts succeed, some fail)
  - Test: Returns 400 for missing Jira config
  - Test: Returns 401 for Jira auth failure
  - Mock Jira attachment upload and comment creation
  - Verify response matches PushResponse schema

- [X] **T007** [P] Contract test for GET /jira endpoint in `components/backend/handlers_test.go`
  - Test: getSessionJiraLinks returns 200 with links array
  - Test: Returns empty array for session with no links
  - Test: Returns 404 for non-existent session
  - Verify response matches JiraLinksResponse schema

- [X] **T008** [P] Integration test for artifact push workflow in `components/backend/integration_test.go`
  - Scenario: Create AgenticSession → List artifacts → Validate issue → Push artifacts → Verify annotations updated
  - Use Kind/K8s test cluster or mock Kubernetes client
  - Mock Jira API calls
  - Verify JiraLink annotation added to AgenticSession CR

- [X] **T009** [P] Integration test for error scenarios in `components/backend/integration_test.go`
  - Test: Missing Jira config (secret not found)
  - Test: Invalid Jira token (401 from Jira)
  - Test: Artifact too large (>10MB)
  - Test: Network timeout to Jira
  - Verify error codes and retryable flags

---

## Phase 3.3: Core Implementation (ONLY after tests are failing)

### Backend (Go)

- [X] **T010** Implement Jira client wrapper in `components/backend/jira_client.go`
  - Function: loadJiraConfig(namespace) - reads runner secret
  - Function: validateIssue(config, issueKey) - GET /rest/api/2/issue/{issueKey}
  - Function: uploadAttachment(config, issueKey, filepath, reader) - POST with multipart/form-data
  - Function: createComment(config, issueKey, body) - POST /rest/api/2/issue/{issueKey}/comment
  - HTTP client with 30s timeout, exponential backoff for retries
  - Error mapping to ErrorResponse codes

- [X] **T011** Implement helper functions for JiraLink annotations in `components/backend/jira_helpers.go`
  - Function: getJiraLinks(cr) - parse jiraLinks annotation from AgenticSession
  - Function: addJiraLink(cr, link) - append JiraLink to annotation array
  - Function: updateSessionAnnotations(ctx, client, cr) - update Kubernetes CR
  - JSON marshaling/unmarshaling with error handling

- [X] **T012** Implement listSessionArtifacts handler in `components/backend/jira_handlers.go`
  - Get AgenticSession CR, extract status.stateDir
  - List files in stateDir (filesystem or PVC access)
  - Build SessionArtifact array with path, size, mimeType, lastModified
  - Validate artifact paths (no path traversal)
  - Return ArtifactListResponse

- [X] **T013** Implement validateSessionJiraIssue handler in `components/backend/jira_handlers.go`
  - Parse ValidateIssueRequest
  - Validate issue key format (regex)
  - Load Jira config from runner secret
  - Call jiraClient.validateIssue()
  - Return ValidateIssueResponse with valid flag and issue metadata

- [X] **T014** Implement pushSessionToJira handler in `components/backend/jira_handlers.go`
  - Parse PushRequest
  - Load AgenticSession CR and Jira config
  - Load artifacts from stateDir
  - Validate artifact sizes (≤10MB each)
  - Upload artifacts in parallel (max 5 concurrent, use goroutines + sync.WaitGroup)
  - Create Jira comment with session metadata
  - Update AgenticSession annotations with JiraLinks
  - Return PushResponse with success/failure per artifact

- [X] **T015** Implement getSessionJiraLinks handler in `components/backend/jira_handlers.go`
  - Load AgenticSession CR
  - Parse jiraLinks annotation using jira_helpers.getJiraLinks()
  - Sort links by timestamp (most recent first)
  - Return JiraLinksResponse

- [X] **T016** Add error handling middleware in `components/backend/jira_handlers.go`
  - Centralized error response formatting (ErrorResponse struct)
  - Map Go errors to ErrorResponse codes
  - Set retryable flag based on error type
  - Log errors with request context (user, session, operation)

### Frontend (TypeScript/React)

- [ ] **T017** [P] Create PushToJiraDialog component in `components/frontend/src/components/PushToJiraDialog.tsx`
  - Props: sessionName, projectName, isOpen, onClose
  - State: issueKey, selectedArtifacts, isValidating, isPushing, validationResult, pushResult
  - UI: Issue key input, "Validate Issue" button, artifact checklist, "Push" button (disabled until validated)
  - Display issue metadata after validation (title, status, project)
  - Show upload progress during push
  - Display success message with Jira link, or error messages
  - Use React hooks (useState, useEffect)

- [ ] **T018** [P] Create JiraLinkBadge component in `components/frontend/src/components/JiraLinkBadge.tsx`
  - Props: jiraKey, jiraUrl
  - Render clickable badge: "Linked to Jira: PROJ-123"
  - Link opens Jira issue in new tab
  - Styling: compact badge with Jira icon

- [ ] **T019** [P] Create PushHistoryView component in `components/frontend/src/components/PushHistoryView.tsx`
  - Props: jiraLinks (JiraLink array)
  - Render table with columns: Issue Key, Artifacts, Timestamp, Status
  - Sort by timestamp (most recent first)
  - Status indicator: success (green checkmark), failed (red X with error tooltip)
  - Retry button for failed pushes

- [ ] **T020** [P] Create API client functions in `components/frontend/src/lib/jiraApi.ts`
  - Function: listArtifacts(projectName, sessionName) - GET /artifacts
  - Function: validateIssue(projectName, sessionName, issueKey) - POST /jira/validate
  - Function: pushToJira(projectName, sessionName, request) - POST /jira
  - Function: getJiraLinks(projectName, sessionName) - GET /jira
  - Use fetch with Bearer token, error handling, TypeScript types

- [ ] **T021** Integrate PushToJiraDialog into session detail page in `components/frontend/src/app/projects/[projectName]/sessions/[sessionName]/page.tsx`
  - Add "Push to Jira" button (visible when phase is Completed/Failed/Stopped)
  - Conditionally disable button if no Jira config (check via API or feature flag)
  - Open PushToJiraDialog on button click
  - Fetch Jira links on page load
  - Display JiraLinkBadge if links exist
  - Add expandable PushHistoryView section

- [ ] **T022** [P] Create component tests for PushToJiraDialog in `components/frontend/src/components/__tests__/PushToJiraDialog.test.tsx`
  - Test: Renders with issue key input and artifact checklist
  - Test: Validation button calls validateIssue API
  - Test: Push button enabled after successful validation
  - Test: Displays error message on validation failure
  - Test: Shows progress indicator during push
  - Use React Testing Library and Jest

- [ ] **T023** [P] Create component tests for JiraLinkBadge in `components/frontend/src/components/__tests__/JiraLinkBadge.test.tsx`
  - Test: Renders with Jira key
  - Test: Link opens in new tab
  - Test: Correct href format

---

## Phase 3.4: Integration

- [ ] **T024** Add E2E test for full push workflow in `components/frontend/e2e/jira-push.spec.ts`
  - Scenario: Navigate to session detail → Click "Push to Jira" → Enter issue key → Validate → Select artifacts → Push → Verify badge appears
  - Mock backend API responses
  - Use Playwright or Cypress
  - Verify success toast/notification displayed

- [ ] **T025** Add E2E test for error scenarios in `components/frontend/e2e/jira-errors.spec.ts`
  - Test: Missing Jira config error display
  - Test: Invalid issue key validation error
  - Test: Network error with retry prompt
  - Verify error messages match ErrorResponse.code descriptions

- [ ] **T026** Validate Jira API rate limiting in `components/backend/jira_client.go`
  - Add exponential backoff for 429 responses
  - Log rate limit headers (X-RateLimit-Remaining, X-RateLimit-Reset)
  - Test with mock Jira API returning 429

- [ ] **T027** Add logging for Jira operations in `components/backend/handlers.go`
  - Log: timestamp, user, session name, issue key, operation (validate/push), success/failure, duration
  - Include request IDs for correlation
  - Use structured logging (e.g., logrus or zap)

---

## Phase 3.5: Polish

- [ ] **T028** [P] Add unit tests for jiraClient functions in `components/backend/jira_client_test.go`
  - Test: loadJiraConfig handles missing secret
  - Test: validateIssue handles 404 response
  - Test: uploadAttachment multipart encoding
  - Test: createComment request body format
  - Test: Exponential backoff logic

- [ ] **T029** [P] Add unit tests for jiraHelpers functions in `components/backend/jira_helpers_test.go`
  - Test: getJiraLinks parses annotation correctly
  - Test: addJiraLink appends to existing array
  - Test: JSON marshaling edge cases (empty array, special characters)

- [ ] **T030** [P] Add performance test for artifact upload in `components/backend/handlers_test.go`
  - Test: Upload 5 artifacts (5MB total) completes in <3s
  - Test: Parallel uploads faster than sequential
  - Use httptest.Server with simulated latency

- [ ] **T031** [P] Update vTeam user documentation in `components/docs/user-guide/jira-integration.md`
  - Section: Configuring Jira credentials (runner secrets)
  - Section: Pushing artifacts to Jira (step-by-step with screenshots)
  - Section: Viewing push history
  - Section: Troubleshooting common errors

- [ ] **T032** [P] Update vTeam API documentation in `components/docs/api/session-jira.md`
  - Document 4 new endpoints with request/response examples
  - Include error codes and retry guidance
  - Link to OpenAPI spec (contracts/session-jira-api.yaml)

- [ ] **T033** Update vTeam CLAUDE.md in `/workspace/sessions/agentic-session-1760033087/workspace/vTeam/CLAUDE.md`
  - Add Jira integration to "Key Features" section
  - Add recent changes entry: "Added session artifact push to Jira"
  - Add tech context: Jira REST API v2, runner secrets pattern
  - Keep under 150 lines (prune old content if needed)

- [ ] **T034** [P] Refactor duplicate code in handlers
  - Extract common logic: loadSession, loadJiraConfig, errorResponse
  - Consolidate Kubernetes client initialization
  - Apply DRY principle to handler functions

- [ ] **T035** Run quickstart validation in `specs/001-develop-a-new/quickstart.md`
  - Execute all 11 test scenarios manually or via script
  - Verify acceptance criteria pass
  - Document any failures or deviations

- [ ] **T036** Code review and final validation
  - Review all code against constitution principles (constitution.md)
  - Verify no hardcoded secrets or credentials
  - Check error handling completeness
  - Ensure logging doesn't expose sensitive data
  - Confirm all tests pass (contract, unit, integration, E2E)
  - Verify performance targets met (<3s upload, <500ms validation)

---

## Dependencies

```
Setup (T001-T003)
  ↓
Tests (T004-T009) ← MUST FAIL before implementation
  ↓
Backend Core (T010-T016)
  ├─ T010 (jira_client.go) → blocks T013, T014
  ├─ T011 (jira_helpers.go) → blocks T014, T015
  └─ T012-T015 (handlers.go) - sequential (same file)
  ↓
Frontend Core (T017-T023) - can run parallel with backend
  ├─ T017-T019 (components) [P]
  ├─ T020 (API client) [P]
  ├─ T021 (integration)
  └─ T022-T023 (component tests) [P]
  ↓
Integration (T024-T027)
  ↓
Polish (T028-T036)
  ├─ T028-T032 (tests, docs) [P]
  └─ T033-T036 (final tasks)
```

### Key Blocking Relationships
- **T010 blocks T013, T014**: jira_client.go must exist before handlers call it
- **T011 blocks T014, T015**: jira_helpers.go must exist before annotation operations
- **T012-T016 sequential**: All modify handlers.go (same file)
- **T020 blocks T021**: API client needed for page integration
- **T024-T027 block T035**: Integration must work before quickstart validation
- **All implementation blocks T036**: Final review comes last

---

## Parallel Execution Examples

### Launch contract tests in parallel (T004-T007):
```bash
# All tests in same file (handlers_test.go) but different test functions
go test -v -run TestListSessionArtifacts &
go test -v -run TestValidateSessionJiraIssue &
go test -v -run TestPushSessionToJira &
go test -v -run TestGetSessionJiraLinks &
wait
```

### Launch frontend components in parallel (T017-T019):
```
Task agent 1: "Create PushToJiraDialog component in components/frontend/src/components/PushToJiraDialog.tsx"
Task agent 2: "Create JiraLinkBadge component in components/frontend/src/components/JiraLinkBadge.tsx"
Task agent 3: "Create PushHistoryView component in components/frontend/src/components/PushHistoryView.tsx"
```

### Launch documentation updates in parallel (T031-T032):
```
Task agent 1: "Update user guide in components/docs/user-guide/jira-integration.md"
Task agent 2: "Update API docs in components/docs/api/session-jira.md"
```

---

## Notes

- **[P] tasks** can run in parallel (different files, no dependencies)
- **Verify tests fail** before implementing (T004-T009 must fail initially)
- **Commit after each task** for granular history
- **Backend handlers sequential** (T012-T016) because they modify same file
- **Frontend components parallel** (T017-T019) because they're separate files
- **Repository root** is `/workspace/sessions/agentic-session-1760033087/workspace/vTeam` (not vteam-rfes)

---

## Task Generation Validation

✅ **All contracts have tests**: 4 endpoints → 4 contract test tasks (T004-T007)
✅ **All entities have models**: 9 entities → type definitions in T001-T002
✅ **Tests before implementation**: Phase 3.2 blocks Phase 3.3
✅ **Parallel tasks independent**: [P] tasks modify different files
✅ **File paths specified**: Every task includes exact file path
✅ **No [P] conflicts**: Same-file tasks (T012-T016) are sequential

---

## Success Criteria

This feature is complete when:
1. ✅ All 36 tasks marked complete
2. ✅ All tests pass (contract, unit, integration, E2E)
3. ✅ All 11 quickstart scenarios validated (T035)
4. ✅ Performance targets met: <3s upload, <500ms validation (T030)
5. ✅ Code review passed (T036)
6. ✅ User documentation complete (T031)
7. ✅ No TODOs or FIXMEs in code

**Estimated effort**: 28-36 tasks as predicted in plan.md Phase 2
**Parallel opportunities**: 12 [P] tasks across frontend, tests, and docs
**Critical path**: Setup → Tests → Backend → Integration → Polish (sequential phases)
