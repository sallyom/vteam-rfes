# Research: Jira-Session Artifact Integration

**Date**: 2025-10-09
**Status**: Complete

## Overview
Research findings for implementing Jira integration for AgenticSession artifacts in vTeam. This builds on existing RFEWorkflow Jira integration patterns.

---

## 1. Existing Jira Infrastructure Analysis

### Decision: Reuse existing Jira patterns from RFEWorkflow
**Rationale**: vTeam already has working Jira integration for RFEWorkflow CRs (handlers.go:2879-3141). This provides proven patterns for authentication, API calls, and error handling.

**Key Findings**:
- **Authentication**: Project-scoped runner secrets (JIRA_URL, JIRA_PROJECT, JIRA_API_TOKEN) stored in Kubernetes Secret
- **API Pattern**: Direct HTTP calls to Jira REST API v2 (`/rest/api/2/issue/...`)
- **Storage Pattern**: JiraLinks array in CR annotations/spec with `{path, jiraKey}` structure
- **Error Handling**: HTTP client with 30s timeout, structured error responses

**Implementation Location**:
- Backend: `components/backend/handlers.go` (publishWorkflowFileToJira, getWorkflowJira)
- Frontend: `components/frontend/src/app/api/projects/[name]/rfe-workflows/[id]/jira/route.ts`

**Alternatives Considered**:
- ❌ Atlassian MCP Server: Adds dependency complexity, overkill for simple file uploads
- ❌ Third-party Jira Go client: Existing direct HTTP approach is lightweight and working
- ✅ Direct REST API calls: Matches existing codebase pattern, minimal dependencies

---

## 2. AgenticSession CRD Structure

### Decision: Store Jira links in AgenticSession metadata annotations
**Rationale**: Annotations allow storing arbitrary metadata without CRD schema changes. Follows Kubernetes best practices for extensibility.

**CRD Analysis** (agenticsessions-crd.yaml):
- **Spec Fields**: repos, prompt, llmSettings, timeout, interactive, displayName
- **Status Fields**: phase, message, stateDir (artifact location), result, usage, cost
- **No existing jiraLinks**: Need to add via annotations pattern

**Storage Format** (following RFEWorkflow pattern):
```yaml
metadata:
  annotations:
    vteam.ambient-code/jiraLinks: '[{"path":"transcript.txt","jiraKey":"PROJ-123","timestamp":"2025-10-09T10:00:00Z"}]'
```

**Alternatives Considered**:
- ❌ Add jiraLinks to CRD spec: Requires CRD migration, unnecessary for optional feature
- ❌ External database: Adds infrastructure complexity, loses Kubernetes-native benefits
- ✅ Annotations: Kubernetes-native, no schema changes, queryable via kubectl

---

## 3. Session Artifact Discovery

### Decision: Read artifacts from stateDir path in AgenticSession status
**Rationale**: stateDir is already populated by the runner and contains all session outputs. No need for separate artifact tracking.

**StateDir Structure** (observed from runner behavior):
```
/vteam/sessions/{namespace}/{session-name}/
├── transcript.txt          # Full conversation history
├── result_summary.txt      # Final result
├── tool_outputs/           # Tool execution logs
├── state_snapshot.json     # Session state
└── errors.log             # Error details (if failed)
```

**Artifact Types to Support**:
1. **transcript.txt**: Full chat history (interactive sessions)
2. **result_summary.txt**: Final output summary
3. **tool_outputs/**: Tool execution logs and outputs
4. **state_snapshot.json**: Session state for debugging
5. **errors.log**: Error details for failed sessions

**API Flow**:
1. Backend handler receives push request with session name
2. Load AgenticSession CR to get status.stateDir
3. List files in stateDir using filesystem or PVC access
4. Return file list to frontend for user selection

**Alternatives Considered**:
- ❌ Enumerate artifacts in CR status: Adds overhead, stateDir is sufficient
- ❌ Real-time artifact tracking: Unnecessary complexity for manual push workflow
- ✅ On-demand filesystem scan: Simple, accurate, no state synchronization issues

---

## 4. Jira API Integration

### Decision: Use Jira REST API v2 with multipart/form-data for attachments
**Rationale**: Matches existing RFEWorkflow pattern and supports Jira Cloud/Server compatibility.

**Required API Endpoints**:

#### 4.1 Validate Issue
```
GET /rest/api/2/issue/{issueKey}
Authorization: Bearer {JIRA_API_TOKEN}
```
**Purpose**: Verify issue exists and user has access before upload
**Response**: 200 OK with issue metadata, 404 if not found, 403 if no access

#### 4.2 Upload Attachment
```
POST /rest/api/2/issue/{issueKey}/attachments
Authorization: Bearer {JIRA_API_TOKEN}
X-Atlassian-Token: no-check
Content-Type: multipart/form-data
```
**Purpose**: Upload artifact files to Jira issue
**Limits**: 10MB per file (Jira Cloud), up to 100MB total per issue

#### 4.3 Add Comment
```
POST /rest/api/2/issue/{issueKey}/comment
Authorization: Bearer {JIRA_API_TOKEN}
Content-Type: application/json
Body: {"body": "Session summary..."}
```
**Purpose**: Add session metadata comment with link back to vTeam

**Comment Format**:
```markdown
🤖 vTeam AgenticSession: [session-name]

**Prompt**: {truncated prompt}
**Model**: {llmSettings.model}
**Phase**: {status.phase}
**Duration**: {completionTime - startTime}
**Cost**: ${total_cost_usd}
**Turns**: {num_turns}

[View Session in vTeam](https://vteam.example.com/projects/{ns}/sessions/{name})

Artifacts attached: transcript.txt, result_summary.txt, tool_outputs.zip
```

**Rate Limiting**: Jira Cloud: 300-1000 req/min per token, implement exponential backoff

**Alternatives Considered**:
- ❌ Jira API v3: Not universally supported in Server/Data Center
- ❌ Base64 inline attachments: 10MB limit makes this impractical
- ✅ REST API v2 multipart: Widely supported, handles large files

---

## 5. Authentication & Authorization

### Decision: Reuse project-scoped runner secrets pattern
**Rationale**: Consistent with existing vTeam multi-tenant architecture and RFEWorkflow Jira integration.

**Configuration Storage**:
- **Secret Name**: Determined by ProjectSettings CR (`spec.runnerSecretsName`), defaults to `ambient-runner-secrets`
- **Namespace**: Same as AgenticSession (project namespace)
- **Required Keys**:
  - `JIRA_URL`: Jira instance URL (https://company.atlassian.net or https://jira.company.com)
  - `JIRA_PROJECT`: Default project key (e.g., "VTEAM")
  - `JIRA_API_TOKEN`: API token for authentication

**Access Control**:
- User must have project edit/admin role to push to Jira (reuse existing RBAC)
- Backend validates user token via getK8sClientsForRequest() before Jira operations
- Jira permissions enforced by Jira API (403 if user lacks issue update access)

**Security Considerations**:
- API token stored in Kubernetes Secret (encrypted at rest)
- Token scoped to project, not global
- Backend logs all Jira operations for audit trail

**Alternatives Considered**:
- ❌ User-specific Jira tokens: Breaks service account automation, complex key management
- ❌ Global Jira config: Violates multi-tenant isolation
- ✅ Project-scoped runner secrets: Consistent with vTeam architecture

---

## 6. Frontend UI/UX Patterns

### Decision: Modal dialog with artifact checklist, similar to RFEWorkflow publish flow
**Rationale**: Familiar pattern for users, supports progressive disclosure and validation feedback.

**UI Components to Create**:

#### 6.1 "Push to Jira" Button
- **Location**: Session detail page, visible when phase is Completed/Failed/Stopped
- **Trigger**: Opens push dialog
- **State**: Disabled if no Jira config exists

#### 6.2 Push Dialog
- **Issue Key Input**: Text field with validation (format: PROJECT-123)
- **Artifact Selection**: Checklist of available artifacts (default: all checked)
  - Show file sizes next to each artifact
  - Warn if any file >10MB
- **Validation Button**: "Validate Issue" to check Jira connectivity before push
- **Submit Button**: "Push to Jira" (disabled until validation passes)
- **Progress Indicator**: Upload progress with file names
- **Error Display**: Specific error messages (network, auth, permissions)

#### 6.3 Jira Link Badge
- **Location**: Session list and detail views
- **Content**: "Linked to Jira: PROJ-123, PROJ-456" (clickable links)
- **Hover**: Show push timestamps

#### 6.4 Push History View
- **Location**: Session detail page, expandable section
- **Content**: Table with columns: Issue Key, Artifacts, Timestamp, Status
- **Actions**: Retry failed pushes, view Jira issue

**TypeScript Types** (components/frontend/src/types/agentic-session.ts):
```typescript
interface JiraLink {
  path: string
  jiraKey: string
  timestamp: string
  status: 'success' | 'failed'
  error?: string
}

interface SessionArtifact {
  path: string
  size: number
  mimeType: string
}
```

**Alternatives Considered**:
- ❌ Inline push form on session page: Clutters UI for infrequent action
- ❌ Automatic push on completion: Users need control over when/what to push
- ✅ Modal dialog: Clear workflow, focused interaction, matches RFE pattern

---

## 7. Backend API Design

### Decision: RESTful API following existing handler patterns in handlers.go
**Rationale**: Consistency with existing vTeam API structure and authentication flow.

**New Endpoints**:

#### 7.1 List Artifacts
```
GET /api/projects/:projectName/sessions/:sessionName/artifacts
Authorization: Bearer {user-token}
Response: {
  "artifacts": [
    {"path": "transcript.txt", "size": 12345, "mimeType": "text/plain"},
    {"path": "result_summary.txt", "size": 5678, "mimeType": "text/plain"}
  ]
}
```

#### 7.2 Validate Jira Issue
```
POST /api/projects/:projectName/sessions/:sessionName/jira/validate
Body: {"issueKey": "PROJ-123"}
Response: {
  "valid": true,
  "issue": {"key": "PROJ-123", "summary": "Issue title", "status": "In Progress"}
}
```

#### 7.3 Push Artifacts to Jira
```
POST /api/projects/:projectName/sessions/:sessionName/jira
Body: {
  "issueKey": "PROJ-123",
  "artifacts": ["transcript.txt", "result_summary.txt"]
}
Response: {
  "success": true,
  "jiraKey": "PROJ-123",
  "attachments": ["transcript.txt", "result_summary.txt"],
  "commentId": "10001"
}
```

#### 7.4 Get Jira Links
```
GET /api/projects/:projectName/sessions/:sessionName/jira
Response: {
  "links": [
    {"path": "transcript.txt", "jiraKey": "PROJ-123", "timestamp": "2025-10-09T10:00:00Z"}
  ]
}
```

**Handler Function Naming**:
- `listSessionArtifacts(c *gin.Context)`
- `validateSessionJiraIssue(c *gin.Context)`
- `pushSessionToJira(c *gin.Context)`
- `getSessionJiraLinks(c *gin.Context)`

**Error Response Format**:
```json
{
  "error": "Permission denied",
  "details": "User lacks 'Add Attachments' permission for issue PROJ-123",
  "code": "JIRA_PERMISSION_DENIED"
}
```

**Alternatives Considered**:
- ❌ GraphQL API: Inconsistent with existing REST patterns
- ❌ WebSocket for progress: HTTP SSE simpler for one-way progress updates
- ✅ REST API with structured errors: Matches existing patterns, simple client integration

---

## 8. Error Handling & Resilience

### Decision: Graceful degradation with retry support and detailed error messages
**Rationale**: Jira API calls are external dependencies prone to transient failures. Users need actionable error feedback.

**Error Categories**:

#### 8.1 Configuration Errors (400)
- Missing Jira credentials in runner secrets
- Invalid JIRA_URL format
- **Action**: Link to Project Settings → Runner Secrets

#### 8.2 Validation Errors (400)
- Invalid Jira issue key format
- Artifact size exceeds 10MB
- **Action**: Display specific error, allow user to correct

#### 8.3 Authentication Errors (401/403)
- Invalid JIRA_API_TOKEN
- User lacks vTeam project permissions
- User lacks Jira issue permissions
- **Action**: Show specific permission needed, suggest admin contact

#### 8.4 Network Errors (502/504)
- Jira instance unreachable
- Request timeout (>30s)
- **Action**: Allow retry with exponential backoff

#### 8.5 Rate Limit Errors (429)
- Jira API rate limit exceeded
- **Action**: Show wait time, auto-retry after delay

#### 8.6 Partial Failures
- Some artifacts uploaded, others failed
- **Action**: Show success/failure per artifact, allow retry for failed only

**Retry Strategy**:
- Max 3 retries for network/timeout errors
- Exponential backoff: 1s, 2s, 4s
- No retry for auth/validation errors (require user correction)

**Logging**:
- Backend logs all Jira operations with: timestamp, user, session, issue key, success/failure
- Include request IDs for correlation with Jira logs

**Alternatives Considered**:
- ❌ Silent failure with background retry: Poor user experience, lost context
- ❌ Fail entire operation on first error: Frustrating for large artifact sets
- ✅ Granular error reporting with retry: Best user experience, debugging support

---

## 9. Testing Strategy

### Decision: Contract tests for Jira API, integration tests for end-to-end flow
**Rationale**: External API requires contract validation. Full integration ensures vTeam components work together.

**Test Layers**:

#### 9.1 Contract Tests (Backend)
- **Scope**: Validate Jira REST API v2 contracts
- **Tool**: Go testing with mock HTTP server
- **Coverage**:
  - Issue validation endpoint response formats
  - Attachment upload multipart encoding
  - Comment creation request/response
  - Error response structures (404, 403, 429)

#### 9.2 Unit Tests (Backend)
- **Scope**: Handler logic, error handling, annotation parsing
- **Coverage**:
  - Artifact discovery from stateDir
  - JiraLink annotation serialization/deserialization
  - Configuration validation
  - Error categorization

#### 9.3 Integration Tests (Backend + K8s)
- **Scope**: End-to-end flow with test AgenticSession CR
- **Setup**: Kind cluster or existing K8s test environment
- **Coverage**:
  - Create session, push artifacts, verify annotations updated
  - Missing config error handling
  - Invalid issue key handling
  - Artifact size limit validation

#### 9.4 Component Tests (Frontend)
- **Tool**: React Testing Library
- **Coverage**:
  - Push dialog rendering and validation
  - Artifact selection behavior
  - Error message display
  - Jira link badge rendering

#### 9.5 E2E Tests (Frontend + Backend)
- **Tool**: Playwright or Cypress
- **Coverage**:
  - Full push workflow: open dialog, select artifacts, validate, push, verify badge
  - Error scenarios: missing config, invalid issue, network failure

**Test Data**:
- Mock Jira responses using recorded API fixtures
- Test AgenticSession CRs with known artifact structures
- Runner secrets with valid/invalid Jira configurations

**Alternatives Considered**:
- ❌ Manual testing only: Regression risk, time-consuming
- ❌ Only unit tests: Misses integration issues with K8s/Jira
- ✅ Multi-layer testing: Comprehensive coverage, fast feedback

---

## 10. Performance Considerations

### Decision: Streaming uploads for large files, parallel attachment uploads
**Rationale**: Session artifacts can be large (10MB limit). Efficient transfers improve UX.

**Optimization Strategies**:

#### 10.1 File Uploads
- **Streaming**: Use io.Reader to stream files from PVC to Jira (avoid loading in memory)
- **Parallel**: Upload multiple attachments concurrently (max 5 parallel)
- **Compression**: Optionally zip multiple small files before upload
- **Progress**: Report upload progress to frontend via chunked responses or SSE

#### 10.2 Artifact Discovery
- **Caching**: Cache artifact list for 60s to reduce filesystem scans
- **Lazy Loading**: Only scan stateDir when user requests artifact list

#### 10.3 Jira API Optimization
- **Batch Comments**: Single comment with all session metadata (not multiple comments)
- **Conditional Validation**: Skip re-validation if issue validated <5min ago (client-side cache)
- **Connection Pooling**: Reuse HTTP client for multiple Jira API calls

**Performance Targets** (from Technical Context):
- <3s for complete artifact upload (5 files, 5MB total)
- <500ms for Jira issue validation
- <1s for artifact list retrieval

**Monitoring**:
- Log operation duration for each push operation
- Track Jira API response times
- Alert on >10s push operations

**Alternatives Considered**:
- ❌ Synchronous serial uploads: Slow for multiple artifacts
- ❌ No progress feedback: Poor UX for large uploads
- ✅ Parallel streaming with progress: Optimal speed and UX

---

## 11. Deployment & Configuration

### Decision: No new infrastructure required, feature flag for gradual rollout
**Rationale**: Leverage existing vTeam deployment and Jira configuration in runner secrets.

**Configuration**:
- **Feature Flag**: Environment variable `ENABLE_SESSION_JIRA=true` in backend deployment
- **Jira Credentials**: Per-project runner secrets (existing pattern)
- **UI Toggle**: Show "Push to Jira" button only if backend feature flag enabled

**Deployment Steps**:
1. Deploy backend with new handlers (backwards compatible)
2. Deploy frontend with push dialog UI
3. Enable feature flag in staging environment
4. Validate with test sessions
5. Enable in production environment
6. Document in user guide

**Backwards Compatibility**:
- Old AgenticSessions without Jira links continue to work
- No CRD schema changes (annotations only)
- Gradual opt-in per project (via runner secret configuration)

**Alternatives Considered**:
- ❌ Separate Jira service: Over-engineered for simple integration
- ❌ Always-on feature: Risk for untested edge cases
- ✅ Feature flag with gradual rollout: Safe, reversible, testable

---

## Summary of Key Decisions

| Area | Decision | Rationale |
|------|----------|-----------|
| **Architecture** | Extend existing RFEWorkflow Jira pattern | Proven, minimal new code |
| **Storage** | Annotations for JiraLinks | No CRD changes, Kubernetes-native |
| **Artifacts** | Read from stateDir on-demand | Simple, accurate, no state sync |
| **Jira API** | REST API v2 multipart uploads | Widely supported, handles large files |
| **Auth** | Project-scoped runner secrets | Consistent with vTeam multi-tenancy |
| **UI** | Modal dialog with validation | Familiar pattern, clear workflow |
| **API** | RESTful handlers in handlers.go | Matches existing patterns |
| **Errors** | Granular errors with retry | Best UX, actionable feedback |
| **Testing** | Multi-layer: contract, unit, integration | Comprehensive, fast feedback |
| **Performance** | Parallel streaming uploads | Fast, scalable, good UX |
| **Deployment** | Feature flag, no new infra | Safe rollout, reversible |

---

## Next Steps (Phase 1: Design)
1. Define data model for JiraLink structure
2. Generate OpenAPI contracts for new endpoints
3. Create contract tests (failing)
4. Write quickstart.md with user workflow
5. Update agent context files
