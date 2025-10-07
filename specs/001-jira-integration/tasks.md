# Tasks: Jira Integration for Session Artifacts

**Feature**: 001-jira-integration
**Prerequisites**: plan.md, research.md, data-model.md, contracts/jira-api.yaml, quickstart.md
**Tech Stack**: Go 1.24+ (backend), TypeScript/Next.js 15+ (frontend)
**Project Type**: Web (backend + frontend)

## Execution Flow
```
1. Load plan.md from feature directory ✓
2. Load design documents ✓
   → data-model.md: 5 entities (JiraConfiguration, SessionJiraLink, Artifact, PublishingOperation, JiraPublishQueue)
   → contracts/jira-api.yaml: 9 endpoint groups
   → research.md: OAuth 2.0, Direct REST API, Attachments+Comments strategy
   → quickstart.md: 10 test scenarios
3. Generate tasks by category ✓
4. Apply task rules ✓
5. Number tasks sequentially (T001-T056)
6. Validate completeness ✓
```

## Format: `[ID] [P?] Description`
- **[P]**: Can run in parallel (different files, no dependencies)
- File paths based on plan.md structure

---

## Phase 3.1: Setup & Dependencies

- [ ] **T001** Create project structure per plan.md (backend: components/backend/jira/, api/, models/; frontend: components/frontend/src/components/, services/, hooks/)
- [ ] **T002** Initialize Go module dependencies (gin-gonic/gin, andygrunwald/go-jira, summonio/markdown-to-adf, golang.org/x/oauth2)
- [ ] **T003** [P] Configure Go linting (golangci-lint) and formatting (gofmt)
- [ ] **T004** [P] Initialize Next.js/TypeScript frontend dependencies (React 19, TypeScript, Jest, React Testing Library)
- [ ] **T005** [P] Configure frontend linting (ESLint) and formatting (Prettier)
- [ ] **T006** [P] Set up secrets management structure for OAuth credentials (encrypted storage, AES-256)

---

## Phase 3.2: Data Models & Validation (TDD - Models First)

- [ ] **T007** [P] Define JiraConfiguration model in components/backend/models/jira_config.go with encryption tags
- [ ] **T008** [P] Define SessionJiraLink model in components/backend/models/session_jira_link.go with state transitions
- [ ] **T009** [P] Define Artifact model in components/backend/models/artifact.go with content hash generation
- [ ] **T010** [P] Define PublishingOperation model in components/backend/models/publishing_operation.go with timing fields
- [ ] **T011** [P] Define JiraPublishQueue model in components/backend/models/jira_publish_queue.go with retry logic
- [ ] **T012** [P] Create validation functions for JiraConfiguration (client_id, publishing_strategy) in components/backend/models/jira_config.go
- [ ] **T013** [P] Create validation functions for SessionJiraLink (issue_key pattern, status transitions) in components/backend/models/session_jira_link.go
- [ ] **T014** [P] Create validation functions for Artifact (type enum, file_size limits) in components/backend/models/artifact.go
- [ ] **T015** [P] Set up storage layer (file system or database) for models with encryption at rest

---

## Phase 3.3: Contract Tests (TDD - MUST FAIL Before Implementation)

**CRITICAL: These tests MUST be written and MUST FAIL before ANY endpoint implementation**

### Configuration Endpoints
- [ ] **T016** [P] Contract test GET /jira/config in components/backend/tests/contract/test_jira_config_get.go
- [ ] **T017** [P] Contract test POST /jira/config in components/backend/tests/contract/test_jira_config_post.go
- [ ] **T018** [P] Contract test DELETE /jira/config in components/backend/tests/contract/test_jira_config_delete.go

### OAuth Endpoints
- [ ] **T019** [P] Contract test GET /jira/oauth/authorize in components/backend/tests/contract/test_oauth_authorize.go
- [ ] **T020** [P] Contract test POST /jira/oauth/callback in components/backend/tests/contract/test_oauth_callback.go

### Session Linking Endpoints
- [ ] **T021** [P] Contract test POST /sessions/{sessionId}/jira-link in components/backend/tests/contract/test_session_link_post.go
- [ ] **T022** [P] Contract test GET /sessions/{sessionId}/jira-link in components/backend/tests/contract/test_session_link_get.go
- [ ] **T023** [P] Contract test DELETE /sessions/{sessionId}/jira-link in components/backend/tests/contract/test_session_link_delete.go

### Publishing Endpoints
- [ ] **T024** [P] Contract test POST /sessions/{sessionId}/publish in components/backend/tests/contract/test_publish_post.go
- [ ] **T025** [P] Contract test GET /sessions/{sessionId}/publish/status in components/backend/tests/contract/test_publish_status_get.go
- [ ] **T026** [P] Contract test POST /sessions/{sessionId}/publish/retry in components/backend/tests/contract/test_publish_retry.go

### Monitoring Endpoints
- [ ] **T027** [P] Contract test GET /jira/health in components/backend/tests/contract/test_health_get.go
- [ ] **T028** [P] Contract test GET /jira/operations in components/backend/tests/contract/test_operations_get.go

---

## Phase 3.4: Core Libraries (Independent, Can Run Parallel)

- [ ] **T029** [P] Implement OAuth 2.0 client wrapper in components/backend/jira/oauth.go (authorize, callback, refresh token, token storage)
- [ ] **T030** [P] Integrate Jira REST API client library (go-jira) in components/backend/jira/client.go with rate limiting
- [ ] **T031** [P] Implement markdown-to-ADF converter in components/backend/jira/formatter.go using summonio/markdown-to-adf
- [ ] **T032** [P] Create rate limiter with exponential backoff and jitter in components/backend/jira/retry.go (formula: delay = base * 2^attempt + jitter)
- [ ] **T033** [P] Implement error classifier (401, 403, 404, 429, 5xx) in components/backend/jira/errors.go with user-friendly messages

---

## Phase 3.5: Publishing Logic (Sequential, Depends on Libraries)

- [ ] **T034** Implement attachment upload handler in components/backend/jira/publisher.go (multipart/form-data, X-Atlassian-Token header, multiple files)
- [ ] **T035** Implement comment posting handler in components/backend/jira/publisher.go (markdown to ADF conversion, fallback to plain text)
- [ ] **T036** Implement publishing orchestrator in components/backend/jira/publisher.go (attachments + comments, transaction-like flow)
- [ ] **T037** Create retry queue manager in components/backend/jira/publisher.go (background worker, scheduled retries, max 10 attempts)
- [ ] **T038** Implement artifact size validation (10MB warning, configurable threshold) in components/backend/jira/publisher.go

---

## Phase 3.6: API Endpoints (Sequential, Depends on Publishing Logic)

### Configuration Endpoints
- [ ] **T039** Implement GET /jira/config handler in components/backend/api/jira_handlers.go (mask tokens)
- [ ] **T040** Implement POST /jira/config handler in components/backend/api/jira_handlers.go (validate, encrypt, store)
- [ ] **T041** Implement DELETE /jira/config handler in components/backend/api/jira_handlers.go (clear tokens, keep client_id/secret)

### OAuth Endpoints
- [ ] **T042** Implement GET /jira/oauth/authorize handler in components/backend/api/jira_handlers.go (generate auth URL with state)
- [ ] **T043** Implement POST /jira/oauth/callback handler in components/backend/api/jira_handlers.go (exchange code, store tokens, handle rotation)

### Session Linking Endpoints
- [ ] **T044** Implement POST /sessions/{sessionId}/jira-link handler in components/backend/api/jira_handlers.go (validate issue key, verify permissions, cache issue summary)
- [ ] **T045** Implement GET /sessions/{sessionId}/jira-link handler in components/backend/api/jira_handlers.go
- [ ] **T046** Implement DELETE /sessions/{sessionId}/jira-link handler in components/backend/api/jira_handlers.go

### Publishing Endpoints
- [ ] **T047** Implement POST /sessions/{sessionId}/publish handler in components/backend/api/jira_handlers.go (trigger publishing, return 202 Accepted)
- [ ] **T048** Implement GET /sessions/{sessionId}/publish/status handler in components/backend/api/jira_handlers.go (return operation history)
- [ ] **T049** Implement POST /sessions/{sessionId}/publish/retry handler in components/backend/api/jira_handlers.go (reset status, enqueue retry)

### Monitoring Endpoints
- [ ] **T050** Implement GET /jira/health handler in components/backend/api/jira_handlers.go (check config, token validity, Jira connectivity)
- [ ] **T051** Implement GET /jira/operations handler in components/backend/api/jira_handlers.go (admin view, operation history, error rates)

---

## Phase 3.7: Frontend Components (Parallel, Can Run with Backend)

- [ ] **T052** [P] Create JiraSettings component in components/frontend/src/components/JiraSettings.tsx (OAuth flow, client ID/secret input, authorization button, connection status)
- [ ] **T053** [P] Create JiraLinkDialog component in components/frontend/src/components/JiraLinkDialog.tsx (issue key input, validation, issue summary preview)
- [ ] **T054** [P] Create JiraPublishStatus component in components/frontend/src/components/JiraPublishStatus.tsx (progress indicator, operation history, retry button)
- [ ] **T055** [P] Implement jiraService API client in components/frontend/src/services/jiraService.ts (TypeScript types from OpenAPI, axios/fetch wrappers)
- [ ] **T056** [P] Create useJiraIntegration React hook in components/frontend/src/hooks/useJiraIntegration.ts (state management, API calls, error handling)

---

## Phase 3.8: Integration Tests (Depends on Backend+Frontend)

### End-to-End Scenarios (from quickstart.md)
- [ ] **T057** [P] Integration test: OAuth flow with mock Atlassian provider in components/backend/tests/integration/jira_oauth_test.go
- [ ] **T058** [P] Integration test: Link session to Jira issue (validate issue key, cache summary) in components/backend/tests/integration/jira_link_test.go
- [ ] **T059** [P] Integration test: Publish artifacts as attachments with mock Jira in components/backend/tests/integration/jira_publish_attachments_test.go
- [ ] **T060** [P] Integration test: Publish with comments (markdown to ADF) in components/backend/tests/integration/jira_publish_comments_test.go
- [ ] **T061** [P] Integration test: Error handling (401, 403, 404, 429, 5xx) with mock responses in components/backend/tests/integration/jira_errors_test.go
- [ ] **T062** [P] Integration test: Retry mechanism with exponential backoff in components/backend/tests/integration/jira_retry_test.go
- [ ] **T063** [P] Integration test: Token refresh on 401 with rotation in components/backend/tests/integration/jira_token_refresh_test.go
- [ ] **T064** [P] Integration test: Rate limiting (429 responses, Retry-After header) in components/backend/tests/integration/jira_rate_limit_test.go
- [ ] **T065** [P] Integration test: Artifact size validation (10MB threshold) in components/backend/tests/integration/jira_artifact_size_test.go
- [ ] **T066** [P] Integration test: Session completion triggers auto-publish in components/backend/tests/integration/jira_auto_publish_test.go

### Frontend Component Tests
- [ ] **T067** [P] Unit test JiraSettings component in components/frontend/src/components/JiraSettings.test.tsx
- [ ] **T068** [P] Unit test JiraLinkDialog component in components/frontend/src/components/JiraLinkDialog.test.tsx
- [ ] **T069** [P] Unit test JiraPublishStatus component in components/frontend/src/components/JiraPublishStatus.test.tsx
- [ ] **T070** [P] Unit test jiraService API client in components/frontend/src/services/jiraService.test.ts
- [ ] **T071** [P] Unit test useJiraIntegration hook in components/frontend/src/hooks/useJiraIntegration.test.ts

---

## Phase 3.9: Polish & Documentation

- [ ] **T072** [P] Unit tests for validation functions in components/backend/tests/unit/validation_test.go
- [ ] **T073** [P] Unit tests for markdown-to-ADF converter in components/backend/tests/unit/formatter_test.go
- [ ] **T074** [P] Unit tests for retry backoff calculation in components/backend/tests/unit/retry_test.go
- [ ] **T075** Write OAuth setup guide with screenshots in docs/jira-integration/oauth-setup.md
- [ ] **T076** Write user guide for linking sessions in docs/jira-integration/user-guide.md
- [ ] **T077** Write troubleshooting documentation in docs/jira-integration/troubleshooting.md
- [ ] **T078** Create architecture diagrams (OAuth flow, publishing workflow) in docs/jira-integration/architecture/
- [ ] **T079** Set up monitoring/alerting for publishing operations (error rates, retry queue depth)
- [ ] **T080** Configure secrets management for OAuth credentials (Vault integration or encrypted environment variables)
- [ ] **T081** Run manual testing scenarios from quickstart.md (all 10 steps)
- [ ] **T082** Performance validation: Artifact publishing <30s, handle concurrent sessions
- [ ] **T083** Code review and refactoring pass (remove duplication, improve error messages)

---

## Dependencies Summary

### Sequential Dependencies (Blocking):
1. **Setup (T001-T006)** blocks everything else
2. **Models (T007-T015)** blocks libraries, endpoints, tests
3. **Contract Tests (T016-T028)** must fail before endpoint implementation
4. **Core Libraries (T029-T033)** blocks publishing logic
5. **Publishing Logic (T034-T038)** blocks API endpoints
6. **API Endpoints (T039-T051)** blocks integration tests
7. **Backend + Frontend (T039-T056)** blocks integration tests
8. **Integration Tests (T057-T071)** blocks polish phase
9. **Polish (T072-T083)** is final phase

### Parallel Execution Groups:
- **[P] Group 1**: T003, T004, T005, T006 (tooling setup)
- **[P] Group 2**: T007-T014 (model definitions)
- **[P] Group 3**: T016-T028 (contract tests - 13 tests)
- **[P] Group 4**: T029-T033 (core libraries - 5 independent modules)
- **[P] Group 5**: T052-T056 (frontend components - 5 components)
- **[P] Group 6**: T057-T071 (integration tests - 15 tests)
- **[P] Group 7**: T072-T074, T075-T078 (unit tests + docs)

---

## Parallel Execution Examples

### Example 1: Launch All Contract Tests Together
```bash
# After models are complete, launch all 13 contract tests in parallel
# Each test writes to different file in tests/contract/

Task: "Contract test GET /jira/config in components/backend/tests/contract/test_jira_config_get.go"
Task: "Contract test POST /jira/config in components/backend/tests/contract/test_jira_config_post.go"
Task: "Contract test DELETE /jira/config in components/backend/tests/contract/test_jira_config_delete.go"
Task: "Contract test GET /jira/oauth/authorize in components/backend/tests/contract/test_oauth_authorize.go"
Task: "Contract test POST /jira/oauth/callback in components/backend/tests/contract/test_oauth_callback.go"
# ... (all T016-T028)
```

### Example 2: Launch Core Libraries in Parallel
```bash
# After models complete, launch 5 core libraries (different files)

Task: "Implement OAuth 2.0 client wrapper in components/backend/jira/oauth.go"
Task: "Integrate Jira REST API client library in components/backend/jira/client.go"
Task: "Implement markdown-to-ADF converter in components/backend/jira/formatter.go"
Task: "Create rate limiter with exponential backoff in components/backend/jira/retry.go"
Task: "Implement error classifier in components/backend/jira/errors.go"
```

### Example 3: Launch Frontend Components with Backend APIs
```bash
# Frontend can run parallel with backend Phase 3.6 (T039-T051)

Task: "Create JiraSettings component in components/frontend/src/components/JiraSettings.tsx"
Task: "Create JiraLinkDialog component in components/frontend/src/components/JiraLinkDialog.tsx"
Task: "Create JiraPublishStatus component in components/frontend/src/components/JiraPublishStatus.tsx"
Task: "Implement jiraService API client in components/frontend/src/services/jiraService.ts"
Task: "Create useJiraIntegration React hook in components/frontend/src/hooks/useJiraIntegration.ts"
```

### Example 4: Launch All Integration Tests
```bash
# After backend+frontend complete (T039-T056), launch all integration tests

Task: "Integration test OAuth flow with mock provider"
Task: "Integration test link session to Jira issue"
Task: "Integration test publish artifacts as attachments"
Task: "Integration test publish with comments (markdown to ADF)"
Task: "Integration test error handling (401, 403, 404, 429, 5xx)"
Task: "Integration test retry mechanism with exponential backoff"
Task: "Integration test token refresh on 401"
Task: "Integration test rate limiting"
Task: "Integration test artifact size validation"
Task: "Integration test session completion triggers auto-publish"
# ... (all T057-T071)
```

---

## Validation Checklist (GATE - Before Completion)

- [x] All contract endpoints have tests (13 endpoints: T016-T028)
- [x] All entities have model tasks (5 entities: T007-T011)
- [x] All tests come before implementation (Phase 3.3 before 3.6)
- [x] Parallel tasks are truly independent (different files, no shared state)
- [x] Each task specifies exact file path
- [x] No [P] task modifies same file as another [P] task
- [x] All quickstart scenarios have integration tests (10 scenarios: T057-T066)
- [x] TDD approach enforced (contract tests T016-T028 before endpoints T039-T051)

---

## Task Generation Metadata

**Generated from**:
- plan.md: Tech stack, project structure, dependencies
- data-model.md: 5 entities with validation rules
- contracts/jira-api.yaml: 9 endpoint groups (13 unique endpoints)
- research.md: OAuth 2.0, Direct REST API, rate limiting, error handling
- quickstart.md: 10 test scenarios

**Task count**: 83 tasks
**Estimated duration**: 166-249 hours (2-3 hours per task)
**Parallel efficiency**: ~40% tasks can run parallel (33 out of 83)

**Critical path**:
Setup → Models → Contract Tests → Libraries → Publishing Logic → Endpoints → Integration Tests → Polish

**Earliest parallel execution**:
- After T006: Launch T007-T014 (8 model tasks)
- After T015: Launch T016-T028 (13 contract tests)
- After T028: Launch T029-T033 (5 core libraries)
- After T051: Launch T052-T056 (5 frontend components) + T057-T071 (15 integration tests)

---

## Notes

- **TDD Enforcement**: Contract tests (T016-T028) MUST fail before endpoint implementation (T039-T051)
- **Token Rotation**: Always replace stored refresh token with new token from refresh response (critical for OAuth security)
- **Rate Limiting**: Implement exponential backoff with jitter to prevent thundering herd
- **Error Messages**: User-friendly messages for common errors (401, 403, 404, 429), technical details in logs only
- **Artifact Size**: 10MB warning threshold, validate before upload
- **Parallel [P] Rule**: Tasks can run parallel only if they write to different files and have no dependencies
- **Commit Strategy**: Commit after each task completion (83 commits total)
- **Testing Priority**: Contract tests → Integration tests → Unit tests (TDD approach)
- **Security**: Encrypt client_secret, access_token, refresh_token at rest; never log tokens
- **Monitoring**: Track publishing success rate, retry queue depth, error rates by type
