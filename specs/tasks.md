# Tasks: Jira Integration Restoration for vTeam Platform

**Input**: Design documents from `/workspace/sessions/agentic-session-1762878140/workspace/artifacts/specs/`
**Prerequisites**: plan.md, spec.md, vTeam codebase structure analyzed

**Branch**: `jira-integration-restoration`

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story. Each user story is independently testable and can be delivered as an incremental MVP.

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1-US6)
- Exact file paths from vTeam codebase included

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure for Jira integration

- [ ] T001 Create Jira package structure in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/`
- [ ] T002 [P] Create frontend Jira types in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/types/jira.ts`
- [ ] T003 [P] Add Jira type definitions to backend in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/types/jira.go`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**CRITICAL**: No user story work can begin until this phase is complete

### Backend Foundation

- [ ] T004 Create Jira REST API v3 client in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/client.go`
  - HTTP client with 30s timeout
  - Basic Auth implementation (email:apiToken)
  - Rate limiter implementation (300 requests/10 seconds)
  - Error handling for common Jira API errors (401, 403, 404, 429, 500)

- [ ] T005 Create Markdown to ADF converter in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/markdown.go`
  - Use goldmark parser for Markdown
  - Generate Atlassian Document Format (ADF) JSON
  - Support headers (h1-h6), lists (ordered/unordered), code blocks, links, bold/italic
  - Fallback to plain text if conversion fails

- [ ] T006 Create Jira credential validation in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/validation.go`
  - Test connection function (GET /rest/api/3/serverInfo)
  - Validate URL format (https:// required)
  - Validate project exists (GET /rest/api/3/project/{projectKey})
  - Return available issue types for project

- [ ] T007 Extend AgenticSession types in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/types/session.go`
  - Add JiraLinksMap type for annotations
  - Add helper methods for parsing/serializing Jira links from annotations

- [ ] T008 Add Jira routes to `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/routes.go`
  - POST /api/projects/:projectName/sessions/:sessionName/jira/upload
  - GET /api/projects/:projectName/sessions/:sessionName/jira/links
  - DELETE /api/projects/:projectName/sessions/:sessionName/jira/links/:artifactPath
  - POST /api/projects/:projectName/jira/test-connection
  - GET /api/projects/:projectName/jira/credentials
  - PUT /api/projects/:projectName/jira/credentials
  - Apply existing authentication middleware to all routes

### Frontend Foundation

- [ ] T009 [P] Create Jira API client in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/services/api/jira.ts`
  - testJiraConnection() - POST to test-connection endpoint
  - uploadArtifactToJira() - POST to upload endpoint
  - updateJiraIssue() - POST to upload with operation="update"
  - getJiraLinks() - GET links for session
  - deleteJiraLink() - DELETE link
  - getJiraCredentials() - GET credentials (masked)
  - updateJiraCredentials() - PUT credentials
  - Use existing client.ts pattern with proper error handling

- [ ] T010 [P] Create Jira query hooks in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/services/queries/use-jira.ts`
  - useJiraLinks(projectName, sessionName) - Query hook for links
  - useUploadToJira() - Mutation hook for upload
  - useUpdateJiraIssue() - Mutation hook for update
  - useDeleteJiraLink() - Mutation hook for delete
  - useTestJiraConnection() - Mutation hook for test
  - useJiraCredentials(projectName) - Query hook for credentials
  - useUpdateJiraCredentials() - Mutation hook for credentials update
  - Proper cache invalidation strategies

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Upload Single Artifact to New Jira Issue (Priority: P1) MVP

**Goal**: Enable users to upload a specification document from Shared Artifacts to create a new Jira issue

**Independent Test**: Upload a single artifact file (e.g., spec.md) from Shared Artifacts and verify a new Jira issue is created with the file content as the description

### Backend Implementation for US1

- [ ] T011 [US1] Create upload handler in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go`
  - Implement UploadArtifactToJira handler
  - Retrieve Jira credentials from Kubernetes Secret ({project}-jira-credentials)
  - Proxy request to content service to read artifact file content
  - Convert Markdown to ADF using markdown converter
  - Call Jira API to create issue
  - Store Jira link in AgenticSession annotation (jira.ambient-code.io/links)
  - Return JiraUploadResponse with issueKey, issueUrl, artifactPath, syncedAt

- [ ] T012 [US1] Add credential retrieval helper in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go`
  - Function to load Kubernetes Secret for project
  - Extract JIRA_URL, JIRA_PROJECT_KEY, JIRA_USER_EMAIL, JIRA_API_TOKEN
  - Handle missing credentials with clear error message

- [ ] T013 [US1] Add GetJiraLinks handler in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go`
  - Retrieve AgenticSession CRD
  - Parse jira.ambient-code.io/links annotation
  - Optionally enrich with issue metadata from Jira (summary, status)
  - Return JiraLinksResponse with array of links

### Frontend Implementation for US1

- [ ] T014 [P] [US1] Create Jira upload dialog component in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-upload-dialog.tsx`
  - Dialog with fields: Summary (pre-filled with filename), Issue Type (dropdown), Labels (text input)
  - Use shadcn/ui Dialog, Input, Button, Label components
  - Call useUploadToJira mutation on submit
  - Show loading state during upload
  - Display success toast with issue key on success
  - Display error toast with actionable message on failure
  - Close dialog on success

- [ ] T015 [US1] Modify artifacts panel in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/artifacts-panel.tsx`
  - Add right-click context menu to artifact files
  - Add "Upload to Jira" submenu with "Create New Issue" option
  - Open JiraUploadDialog when "Create New Issue" clicked
  - Pass artifactPath, projectName, sessionName to dialog

- [ ] T016 [US1] Add Jira badge component in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-badge.tsx`
  - Display issue key (e.g., "PROJ-123") as badge next to file name
  - Green badge for active links
  - Use existing badge styling pattern from vTeam UI

- [ ] T017 [US1] Integrate Jira badges into artifacts panel in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/artifacts-panel.tsx`
  - Fetch Jira links using useJiraLinks hook
  - Display JiraBadge component next to files that have links
  - Map artifactPath to badge display

- [ ] T018 [US1] Add error state handling for missing credentials
  - Check if Jira credentials are configured before showing "Upload to Jira" option
  - Display error message directing user to Project Settings if credentials missing
  - Add link to Project Settings in error message

**Checkpoint**: User Story 1 complete - Users can upload artifacts to Jira and see badges

---

## Phase 4: User Story 2 - View Jira Issue from Artifact (Priority: P1) MVP

**Goal**: Enable users to click on a Jira badge to open the corresponding Jira issue in a new browser tab

**Independent Test**: Sync an artifact using US1, then click the Jira badge and verify it opens the correct Jira issue URL in a new tab

### Frontend Implementation for US2

- [ ] T019 [P] [US2] Add click handler to Jira badge in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-badge.tsx`
  - Make badge clickable
  - Open issueUrl in new browser tab on click (target="_blank")
  - Add cursor-pointer styling

- [ ] T020 [P] [US2] Add tooltip to Jira badge in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-badge.tsx`
  - Use shadcn/ui Tooltip component
  - Display on hover: Issue key, summary, and status
  - Lazy load issue metadata on hover if not already cached
  - Handle loading and error states in tooltip

- [ ] T021 [US2] Add context menu actions for synced files in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/artifacts-panel.tsx`
  - Add "View in Jira" option (opens issue in new tab)
  - Add "Copy Jira Link" option (copies issueUrl to clipboard)
  - Show confirmation toast when link copied
  - Only show these options if file has Jira link

**Checkpoint**: User Story 2 complete - Users can navigate from vTeam to Jira with one click

---

## Phase 5: User Story 4 - Jira Configuration and Validation (Priority: P2)

**Goal**: Enable project administrators to configure Jira connection settings and test the connection

**Independent Test**: Enter Jira credentials in Project Settings, click "Test Connection", and verify the connection status is displayed

**Note**: Implementing US4 before US3 because configuration is needed for end-to-end testing of update operations

### Backend Implementation for US4

- [ ] T022 [US4] Add TestJiraConnection handler in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go`
  - Accept JiraTestConnectionRequest with credentials
  - Create temporary Jira client
  - Call validation.TestConnection()
  - Return success with project name and available issue types, or error message

- [ ] T023 [US4] Add GetJiraCredentials handler in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go`
  - Load credentials from Kubernetes Secret
  - Return JiraCredentialsResponse with URL, projectKey, email, apiTokenSet=true (masked)
  - Handle missing secret gracefully

- [ ] T024 [US4] Add UpdateJiraCredentials handler in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go`
  - Accept JiraCredentials in request body
  - Create or update Kubernetes Secret: {project}-jira-credentials
  - Store JIRA_URL, JIRA_PROJECT_KEY, JIRA_USER_EMAIL, JIRA_API_TOKEN
  - Add labels: app.kubernetes.io/managed-by=vteam, vteam.ambient-code.io/component=jira-integration
  - Return success message

### Frontend Implementation for US4

- [ ] T025 [US4] Create Jira configuration section component in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-settings.tsx`
  - Form with fields: Jira URL, Project Key, User Email, API Token
  - "Test Connection" button (calls useTestJiraConnection mutation)
  - "Save" button (calls useUpdateJiraCredentials mutation)
  - Display connection status (success with project name, or error message)
  - Display available issue types on successful connection
  - Mask API token field (password input type)

- [ ] T026 [US4] Add Jira settings to Project Settings page
  - Integrate JiraSettings component into existing project settings layout
  - Add "Integrations" tab or section if not exists
  - Place Jira configuration under Integrations
  - Use existing settings page patterns from vTeam

- [ ] T027 [US4] Add form validation to Jira settings
  - Validate URL format (https:// required, valid domain)
  - Validate Project Key format (uppercase alphanumeric, e.g., PROJ)
  - Validate email format
  - Show validation errors inline
  - Disable "Test Connection" if validation fails

**Checkpoint**: User Story 4 complete - Administrators can configure and test Jira credentials

---

## Phase 6: User Story 3 - Update Existing Jira Issue (Priority: P2)

**Goal**: Enable users to update an existing Jira issue when an artifact file is modified

**Independent Test**: Sync a file (creating issue), modify the file content, re-upload choosing "Update Existing Issue", and verify the Jira issue is updated without creating a new issue

### Backend Implementation for US3

- [ ] T028 [US3] Extend UploadArtifactToJira handler in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go`
  - Support operation="update" in JiraUploadRequest
  - If update operation, use issueKey from request or lookup from existing link
  - Call Jira API PUT /rest/api/3/issue/{issueKey} to update description
  - Add comment to Jira issue: "Content updated from vTeam at {timestamp}"
  - Update syncedAt timestamp in AgenticSession annotation
  - Handle 404 error if issue was deleted (return broken link status)

- [ ] T029 [US3] Add DeleteJiraLink handler in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go`
  - Remove artifact link from AgenticSession annotation
  - Update annotation without affecting other links
  - Return 204 No Content on success

- [ ] T030 [US3] Add issue existence check to client in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/client.go`
  - Implement GetIssue(issueKey) method
  - GET /rest/api/3/issue/{issueKey}?fields=summary,status,key
  - Return issue metadata or 404 error
  - Use for validating links before update

### Frontend Implementation for US3

- [ ] T031 [US3] Add "Update Existing Issue" option to context menu in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/artifacts-panel.tsx`
  - Show "Update Existing Issue" in "Upload to Jira" submenu for files with existing links
  - Call useUpdateJiraIssue mutation directly (no dialog needed)
  - Show loading state during update
  - Display success toast with issue key on success

- [ ] T032 [US3] Add "Unlink from Jira" option to context menu in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/artifacts-panel.tsx`
  - Show "Unlink from Jira" option for files with existing links
  - Open confirmation dialog before unlinking
  - Call useDeleteJiraLink mutation on confirm
  - Remove badge from UI on success

- [ ] T033 [US3] Handle broken links in badge component in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-badge.tsx`
  - Display red badge for status="broken"
  - Show warning icon and "Issue not found" tooltip
  - Add "Re-link" and "Unlink" actions in context menu for broken links

- [ ] T034 [US3] Add re-link dialog component in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-relink-dialog.tsx`
  - Input field for new Jira issue key
  - Validate issue key format (regex: [A-Z]+-\d+)
  - Call update mutation with new issueKey
  - Update badge on success

**Checkpoint**: User Story 3 complete - Users can update existing issues and manage links

---

## Phase 7: User Story 5 - Sync Status Persistence Across Sessions (Priority: P3)

**Goal**: Ensure Jira sync status persists when sessions are closed and reopened

**Independent Test**: Sync artifacts to Jira, note the badges displayed, close the session, reopen it, and verify the Jira badges are still displayed with correct issue keys

### Backend Implementation for US5

- [ ] T035 [US5] Verify annotation persistence in AgenticSession CRD
  - Confirm jira.ambient-code.io/links annotation persists across session restarts
  - Test session closure and reopening
  - Ensure annotations are not cleared by any cleanup processes

### Frontend Implementation for US5

- [ ] T036 [US5] Add Jira links fetch on session load in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/artifacts-panel.tsx`
  - Call useJiraLinks hook when artifacts panel mounts
  - Display badges immediately when session opens
  - Handle loading state while fetching links

- [ ] T037 [US5] Add real-time sync for multi-user sessions
  - When one user syncs an artifact, other users should see the badge after refresh
  - Leverage existing WebSocket infrastructure if available
  - Otherwise, poll for updates on artifacts panel focus

**Checkpoint**: User Story 5 complete - Jira sync status persists across sessions

---

## Phase 8: User Story 6 - Bulk Upload Multiple Artifacts (Priority: P3)

**Goal**: Enable users to select multiple artifacts and upload them to Jira in a single operation

**Independent Test**: Select multiple files using checkboxes in Shared Artifacts view, click "Upload Selected to Jira", and verify that multiple Jira issues are created

### Backend Implementation for US6

- [ ] T038 [US6] Add batch upload support to handler in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go`
  - Accept array of artifactPaths in request
  - Process uploads sequentially (respect rate limits)
  - Track progress and return partial results if some fail
  - Return BatchUploadResponse with successes and failures

- [ ] T039 [US6] Add retry logic with exponential backoff in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/client.go`
  - Retry on network errors (max 3 attempts)
  - Exponential backoff: 1s, 2s, 4s
  - Log all retry attempts
  - Return error after max attempts exceeded

### Frontend Implementation for US6

- [ ] T040 [P] [US6] Add multi-select UI to artifacts panel in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/artifacts-panel.tsx`
  - Add "Select Multiple" toggle button
  - Show checkboxes next to each file when enabled
  - Track selected files in component state
  - Add "Upload Selected to Jira" button (disabled if no files selected)

- [ ] T041 [US6] Create bulk upload dialog in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-bulk-upload-dialog.tsx`
  - Show list of selected files
  - Common settings: Issue Type, Labels
  - Summary auto-generated from filename for each
  - Progress indicator showing "3 of 5 completed"
  - Display individual success/failure states
  - Allow retry of failed uploads

- [ ] T042 [US6] Add batch upload mutation to queries in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/services/queries/use-jira.ts`
  - useBatchUploadToJira mutation hook
  - Track progress state (completed, total, failures)
  - Invalidate Jira links cache on completion

**Checkpoint**: User Story 6 complete - Users can batch upload multiple artifacts efficiently

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and production readiness

### Testing

- [ ] T043 [P] Add unit tests for Markdown converter in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/markdown_test.go`
  - Test headers (h1-h6), lists, code blocks, links, bold/italic
  - Test edge cases: nested lists, inline code, blockquotes
  - Test fallback to plain text on conversion failure
  - 50+ test cases covering common Markdown elements

- [ ] T044 [P] Add unit tests for Jira client in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/client_test.go`
  - Mock Jira API responses
  - Test authentication (valid/invalid credentials)
  - Test rate limiting behavior
  - Test error handling (401, 403, 404, 429, 500)

- [ ] T045 [P] Add integration tests in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/tests/integration/jira_integration_test.go`
  - Test full upload flow (file to Jira to annotation)
  - Test update flow
  - Test credential validation
  - Requires test Jira instance

- [ ] T046 [P] Add frontend component tests in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/__tests__/`
  - Test JiraUploadDialog rendering and submission
  - Test JiraBadge click and tooltip
  - Test context menu actions
  - Use React Testing Library

### Documentation

- [ ] T047 [P] Create user guide for Jira integration
  - How to configure Jira credentials
  - How to upload artifacts to Jira
  - How to update and manage links
  - Troubleshooting common errors

- [ ] T048 [P] Create admin guide for Jira integration
  - Kubernetes Secret setup
  - Permission requirements
  - Rate limit monitoring
  - Security best practices

### Monitoring & Logging

- [ ] T049 Add structured logging for Jira operations in `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go`
  - Log all Jira API calls with user ID, session, artifact path
  - Log success/failure with response codes
  - Sanitize logs to prevent credential leakage
  - Use existing vTeam logging patterns

- [ ] T050 Add Prometheus metrics for Jira integration
  - Metric: jira_api_requests_total (counter) - labels: method, status_code
  - Metric: jira_api_request_duration_seconds (histogram)
  - Metric: jira_upload_success_total (counter)
  - Metric: jira_upload_failure_total (counter) - labels: error_type
  - Metric: jira_rate_limit_utilization (gauge)

### Security & Error Handling

- [ ] T051 Add input validation and sanitization
  - Validate artifact paths (prevent directory traversal)
  - Sanitize Markdown before conversion (XSS prevention)
  - Validate Jira issue key format (regex: [A-Z]+-\d+)
  - Validate file size limits (reject files >10MB)

- [ ] T052 Add comprehensive error messages
  - Map Jira API errors to user-friendly messages
  - 401: "Authentication failed. Check your Jira credentials in Project Settings."
  - 403: "You don't have permission to create issues in project PROJ. Contact your Jira admin."
  - 404: "Jira issue PROJ-123 not found. It may have been deleted."
  - 429: "Jira rate limit exceeded. Please try again in a few seconds."
  - Network timeout: "Connection to Jira timed out. Check your network and try again."

- [ ] T053 Add credential rotation support
  - Document process for rotating API tokens
  - Ensure no service disruption during rotation
  - Add expiration warnings (if Jira provides token metadata)

### Performance Optimization

- [ ] T054 Implement issue metadata caching
  - Cache issue summary/status for 5 minutes in memory
  - Reduce API calls for badge tooltips by 90%
  - Implement cache invalidation on update

- [ ] T055 Optimize badge rendering performance
  - Lazy load issue status only on badge hover
  - Batch issue metadata fetches (single API call for multiple issues)
  - Use Jira JQL search for bulk metadata retrieval

### Final Validation

- [ ] T056 Run end-to-end tests against test Jira instance
  - Complete user journey: Configure → Upload → View → Update → Unlink
  - Test all user stories independently
  - Verify performance targets (upload <10s, badge render <200ms)

- [ ] T057 Security audit
  - Verify API tokens never appear in logs
  - Verify credentials only accessible via backend
  - Verify RBAC enforcement on all endpoints
  - Test for common vulnerabilities (SSRF, XSS, injection)

- [ ] T058 Performance benchmarking
  - Test bulk upload of 50 files (<2 minutes)
  - Test rate limiter under load
  - Verify Markdown conversion <1s for 50KB files

---

## Dependencies & Execution Order

### Phase Dependencies

1. **Phase 1 (Setup)**: No dependencies - can start immediately
2. **Phase 2 (Foundational)**: Depends on Phase 1 completion - BLOCKS all user stories
3. **Phase 3-8 (User Stories)**: All depend on Phase 2 completion
   - **Recommended order**: US1 → US2 → US4 → US3 → US5 → US6
   - US1 and US2 form the P1 MVP (can deploy after these)
   - US4 enables configuration testing (needed before US3)
   - US3, US5, US6 are P2/P3 enhancements
4. **Phase 9 (Polish)**: Depends on desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: Upload to Jira - Foundation only, no other story dependencies
- **US2 (P1)**: View in Jira - Requires US1 (needs badges to exist)
- **US4 (P2)**: Configuration - Foundation only, no other story dependencies
- **US3 (P2)**: Update Jira - Requires US1 (needs existing links to update)
- **US5 (P3)**: Persistence - Requires US1 (needs links to persist)
- **US6 (P3)**: Bulk Upload - Requires US1 (builds on single upload)

### Within Each User Story

- Backend tasks generally before frontend tasks (APIs must exist for UI to call)
- Exception: Frontend types and API client can be built in parallel with backend [P]
- Models → Services → Handlers → Routes (backend)
- API client → Query hooks → Components (frontend)

### Parallel Opportunities

**Phase 1 (all parallel)**:
- T001, T002, T003 can run in parallel

**Phase 2 (some parallel)**:
- Backend: T004, T005, T006 can run in parallel [P]
- Frontend: T009, T010 can run in parallel [P]
- T007 must complete before T011 (session types needed by handler)
- T008 can run in parallel with others

**Phase 3 (US1)**:
- T014, T016 can run in parallel [P] (different components)
- T011, T012, T013 must be sequential (handler dependencies)
- T015, T017 depend on T014, T016 being complete

**Phase 4 (US2)**:
- T019, T020 can run in parallel [P] (enhancing same component)

**Phase 5 (US4)**:
- T022, T023, T024 must be sequential (backend handlers)
- T025, T026, T027 must be sequential (integrated frontend component)

**Phase 9 (Polish - most parallel)**:
- T043, T044, T045, T046 can all run in parallel [P] (independent test files)
- T047, T048 can run in parallel [P] (independent docs)

---

## Implementation Strategy

### MVP-First Approach (Recommended)

**Minimal Viable Product** = US1 + US2 (P1 stories only)

```
1. Complete Phase 1: Setup (T001-T003)
2. Complete Phase 2: Foundational (T004-T010) ← CRITICAL GATE
3. Complete Phase 3: US1 - Upload (T011-T018)
4. Complete Phase 4: US2 - View (T019-T021)
5. STOP and VALIDATE
   - Test: Upload artifact → See badge → Click badge → Opens Jira
   - Deploy to staging for alpha testing
   - Collect feedback before building P2 features
```

### Incremental Delivery Strategy

After MVP is validated:

```
6. Complete Phase 5: US4 - Configuration (T022-T027)
   - Deploy: Now admins can self-service configuration
7. Complete Phase 6: US3 - Update (T028-T034)
   - Deploy: Now users can update existing issues
8. Complete Phase 7: US5 - Persistence (T035-T037)
   - Deploy: Improved UX with persistent state
9. Complete Phase 8: US6 - Bulk Upload (T038-T042)
   - Deploy: Power user efficiency feature
10. Complete Phase 9: Polish (T043-T058)
    - Production-ready: Full test coverage, monitoring, security audit
```

### Parallel Team Strategy

With 3 developers after Phase 2 completion:

```
Developer A: US1 (T011-T018) → US3 (T028-T034)
Developer B: US2 (T019-T021) → US5 (T035-T037)
Developer C: US4 (T022-T027) → US6 (T038-T042)
All: Phase 9 Polish tasks (many parallel)
```

---

## Checkpoints for Independent Validation

**After Phase 2 (Foundation)**:
- ✓ Jira client can authenticate and make API calls
- ✓ Markdown converter produces valid ADF
- ✓ All API routes are registered
- ✓ Frontend hooks can call backend endpoints

**After US1 (Upload)**:
- ✓ Can upload spec.md from Shared Artifacts to Jira
- ✓ New Jira issue created with correct content
- ✓ Badge appears next to file in artifacts panel
- ✓ Link stored in AgenticSession annotation

**After US2 (View)**:
- ✓ Can click badge to open Jira issue in new tab
- ✓ Tooltip shows issue summary and status on hover
- ✓ "Copy Jira Link" action works

**After US4 (Configuration)**:
- ✓ Can configure Jira credentials in Project Settings
- ✓ "Test Connection" validates credentials
- ✓ Credentials persist and are masked in UI

**After US3 (Update)**:
- ✓ Can update existing Jira issue with modified content
- ✓ Can unlink artifact from Jira
- ✓ Broken links are visually distinguished

**After US5 (Persistence)**:
- ✓ Jira badges persist across session restarts
- ✓ Multiple users see same badges in shared session

**After US6 (Bulk Upload)**:
- ✓ Can select multiple files and upload in batch
- ✓ Progress indicator shows upload status
- ✓ Can retry failed uploads

---

## Notes

- **[P] markers**: Tasks with [P] can run in parallel (different files, no shared state)
- **User story labels**: [US1]-[US6] map tasks to specific user stories for traceability
- **File paths**: All file paths are absolute paths to vTeam codebase
- **Testing philosophy**: Tests in Phase 9 are comprehensive but optional for MVP
- **Commit strategy**: Commit after each task or logical group of parallel tasks
- **Rollback safety**: Each user story can be feature-flagged independently
- **Security**: Never log API tokens; always sanitize error messages

---

## File Path Reference

### Backend Files (Go)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/integration.go` (existing, commented out)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/client.go` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/markdown.go` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/validation.go` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/handlers/jira.go` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/types/jira.go` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/types/session.go` (MODIFY)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/routes.go` (MODIFY)

### Frontend Files (TypeScript/React)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/types/jira.ts` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/services/api/jira.ts` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/services/queries/use-jira.ts` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-upload-dialog.tsx` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-badge.tsx` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-relink-dialog.tsx` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-bulk-upload-dialog.tsx` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/jira-settings.tsx` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/artifacts-panel.tsx` (MODIFY)

### Test Files
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/markdown_test.go` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/backend/jira/client_test.go` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/tests/integration/jira_integration_test.go` (NEW)
- `/workspace/sessions/agentic-session-1762878140/workspace/vTeam/components/frontend/src/components/workspace-sections/__tests__/` (NEW directory)
