# Implementation Plan: Jira Integration Restoration for vTeam Platform

**Branch**: `jira-integration-restoration` | **Date**: 2025-11-11 | **Spec**: [spec.md](/workspace/sessions/agentic-session-1762878140/workspace/artifacts/specs/spec.md)

## Summary

This implementation restores Jira integration capabilities to the vTeam platform, enabling users to upload artifact files from session workspaces to Jira issues, view sync status, and manage the integration lifecycle. The architecture leverages existing session-based infrastructure, Kubernetes secrets for credential storage, and the established content service pattern for file operations. The design prioritizes security through project-scoped credentials, maintains separation of concerns between backend API and frontend UI, and follows the platform's existing architectural patterns for extensibility.

## Technical Context

**Language/Version**: Go 1.21 (backend), TypeScript/React (frontend with Next.js 14)
**Primary Dependencies**:
  - Backend: Gin web framework, Kubernetes client-go, Jira REST API v3
  - Frontend: React 18, TanStack Query, shadcn/ui components

**Storage**:
  - Jira credentials: Kubernetes Secrets (`{project}-jira-credentials`)
  - Jira link metadata: AgenticSession CRD metadata.annotations
  - Session workspace: PersistentVolumeClaim (existing)

**Testing**:
  - Backend: Go testing framework with table-driven tests
  - Frontend: React Testing Library, Jest
  - Integration: E2E tests against test Jira instance

**Target Platform**: Kubernetes/OpenShift cluster (existing vTeam deployment)
**Project Type**: Web application (React frontend + Go backend)
**Performance Goals**:
  - Jira API calls: <5s p95 latency
  - UI responsiveness: <200ms for badge rendering
  - Markdown conversion: <1s for typical spec files (50KB)

**Constraints**:
  - Jira Cloud REST API rate limits: 300 requests/10 seconds per user
  - Session workspace file size: Recommended <10MB per artifact
  - Credentials scope: Project-level (not user-level)

**Scale/Scope**:
  - Support 50+ concurrent projects
  - Handle 1000+ artifacts per project
  - Support both Jira Cloud and Data Center (v3 REST API)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Architecture Principles:**
- ✓ Maintains separation of concerns (backend API, frontend UI, Kubernetes storage)
- ✓ Follows existing patterns (secrets management, content service proxy pattern)
- ✓ Stateless design (no new databases; uses CRD annotations for metadata)
- ✓ Security-first approach (project-scoped credentials, API token encryption)

**Complexity Assessment:**
- ✓ Reuses existing handlers pattern (similar to GitHub integration)
- ✓ Leverages established Kubernetes client patterns
- ✓ Follows content service proxy architecture for file operations
- ✓ No new infrastructure components required

**Justification for Approach:**
The session-based architecture aligns with the platform's existing design philosophy. Storing Jira links as annotations on AgenticSession CRDs provides:
1. Automatic lifecycle management (deleted with session)
2. Native Kubernetes RBAC enforcement
3. No additional database dependencies
4. Auditability through Kubernetes event logs

## Project Structure

### Documentation (this feature)

```text
specs/jira-integration-restoration/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (technical research on Jira API v3)
├── data-model.md        # Phase 1 output (CRD schema, API contracts)
├── quickstart.md        # Phase 1 output (developer setup guide)
├── contracts/           # Phase 1 output (API specs, Jira REST endpoints)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (vTeam repository)

```text
components/backend/
├── handlers/
│   └── jira.go                    # NEW: Jira integration handlers
├── jira/
│   ├── integration.go             # MODIFY: Restore and refactor for sessions
│   ├── client.go                  # NEW: Jira API client wrapper
│   ├── markdown.go                # NEW: Markdown to Jira format converter
│   └── validation.go              # NEW: Credential validation logic
├── types/
│   ├── jira.go                    # NEW: Jira-specific types
│   └── session.go                 # MODIFY: Add jiraLinks to metadata
└── routes.go                      # MODIFY: Add Jira API routes

components/frontend/
├── src/
│   ├── components/
│   │   └── workspace-sections/
│   │       └── artifacts-panel.tsx        # MODIFY: Add Jira badges/actions
│   ├── services/
│   │   ├── queries/
│   │   │   └── use-jira.ts               # NEW: Jira API hooks
│   │   └── api/
│   │       └── jira.ts                    # NEW: Jira API client
│   └── types/
│       └── jira.ts                        # NEW: Frontend Jira types

tests/
├── backend/
│   ├── integration/
│   │   └── jira_integration_test.go       # NEW: Integration tests
│   └── unit/
│       ├── jira_client_test.go            # NEW: Client unit tests
│       └── markdown_converter_test.go     # NEW: Converter tests
└── frontend/
    └── components/
        └── artifacts-panel.test.tsx       # MODIFY: Add Jira badge tests
```

**Structure Decision**: Web application structure selected because vTeam is a full-stack application with distinct React frontend and Go backend layers. The Jira integration touches both layers:
- Backend provides REST API endpoints for Jira operations
- Frontend implements UI components for user interactions
- Both share TypeScript/Go type definitions for consistency

## System Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (React)                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐ │
│  │ Artifacts Panel  │  │ Settings Section │  │ Jira Badge    │ │
│  │  - Upload Menu   │  │  - Jira Config   │  │  - Tooltip    │ │
│  │  - Context Menu  │  │  - Test Conn     │  │  - Click→Open │ │
│  └────────┬─────────┘  └────────┬─────────┘  └───────┬───────┘ │
│           │                     │                     │         │
│           └─────────────────────┼─────────────────────┘         │
│                                 │                               │
└─────────────────────────────────┼───────────────────────────────┘
                                  │ HTTP/REST
┌─────────────────────────────────┼───────────────────────────────┐
│                    Backend API (Go/Gin)                          │
│  ┌─────────────────────────────┴─────────────────────────────┐  │
│  │               Jira Handlers                                │  │
│  │  POST   /api/jira/upload                                   │  │
│  │  POST   /api/jira/test-connection                          │  │
│  │  GET    /api/jira/links/:sessionName                       │  │
│  │  DELETE /api/jira/link/:sessionName/:artifactPath          │  │
│  └────────┬──────────────────────────────────────────┬────────┘  │
│           │                                          │            │
│  ┌────────▼──────────┐  ┌──────────────────────┐   │            │
│  │  Jira Client      │  │  Markdown Converter  │   │            │
│  │  - REST API v3    │  │  - MD → ADF/Wiki     │   │            │
│  │  - Auth Handler   │  │  - Format Detection  │   │            │
│  │  - Rate Limiting  │  │  - Sanitization      │   │            │
│  └────────┬──────────┘  └──────────────────────┘   │            │
│           │                                          │            │
└───────────┼──────────────────────────────────────────┼────────────┘
            │                                          │
            │ HTTP (Basic Auth)                        │ K8s API
            │                                          │
┌───────────▼──────────┐          ┌──────────────────▼──────────┐
│  Jira Cloud/Server   │          │   Kubernetes Cluster         │
│  - REST API v3       │          │  ┌───────────────────────┐  │
│  - Issue Management  │          │  │  Secrets              │  │
│  - Field Validation  │          │  │  - jira-credentials   │  │
│                      │          │  └───────────────────────┘  │
└──────────────────────┘          │  ┌───────────────────────┐  │
                                  │  │  AgenticSession CRD   │  │
                                  │  │  - annotations:       │  │
                                  │  │    jira.links: {...}  │  │
                                  │  └───────────────────────┘  │
                                  └─────────────────────────────┘
```

### Data Flow: Upload Artifact to Jira

```
1. User Action (Frontend)
   ├─> User right-clicks artifact file in Shared Artifacts panel
   ├─> Context menu shows "Upload to Jira" → "Create New Issue"
   └─> Upload dialog opens (pre-filled with file name as summary)

2. API Request (Frontend → Backend)
   POST /api/projects/{project}/sessions/{session}/jira/upload
   Body: {
     artifactPath: "artifacts/spec.md",
     summary: "Feature Specification",
     issueType: "Story",
     labels: ["feature", "spec"],
     description: "Uploaded from vTeam session"
   }

3. Credential Retrieval (Backend)
   ├─> Load Kubernetes Secret: {project}-jira-credentials
   ├─> Extract: jiraUrl, projectKey, userEmail, apiToken
   └─> Validate credentials are present and non-empty

4. File Content Retrieval (Backend)
   ├─> Proxy request to content service:
   │   GET http://ambient-content-{session}.{project}.svc:8080/content/file?path=/sessions/{session}/workspace/artifacts/spec.md
   ├─> Read file content (UTF-8 text)
   └─> Detect format (Markdown, plain text, etc.)

5. Format Conversion (Backend)
   ├─> Parse Markdown structure (headers, lists, code blocks, links)
   ├─> Convert to Jira ADF (Atlassian Document Format) or Wiki markup
   ├─> Sanitize content (remove unsafe HTML, validate links)
   └─> Truncate if exceeds Jira field limits (32KB for Cloud)

6. Jira API Call (Backend → Jira)
   POST https://{jiraUrl}/rest/api/3/issue
   Headers: {
     Authorization: Basic base64({email}:{apiToken}),
     Content-Type: application/json
   }
   Body: {
     fields: {
       project: { key: {projectKey} },
       summary: "Feature Specification",
       description: {ADF content},
       issuetype: { name: "Story" },
       labels: ["feature", "spec"]
     }
   }

7. Metadata Storage (Backend → Kubernetes)
   ├─> Update AgenticSession CRD annotations:
   │   PATCH /apis/vteam.ambient-code/v1alpha1/namespaces/{project}/agenticsessions/{session}
   │   annotations: {
   │     "jira.ambient-code.io/links": "{\"artifacts/spec.md\":{\"issueKey\":\"PROJ-123\",\"issueUrl\":\"...\",\"syncedAt\":\"...\"}}"
   │   }
   └─> Store: artifactPath → {issueKey, issueUrl, syncedAt, status: "active"}

8. Response (Backend → Frontend)
   200 OK
   Body: {
     issueKey: "PROJ-123",
     issueUrl: "https://your-domain.atlassian.net/browse/PROJ-123",
     artifactPath: "artifacts/spec.md"
   }

9. UI Update (Frontend)
   ├─> Refresh artifacts list (re-fetch session metadata)
   ├─> Display Jira badge next to artifact file (badge shows "PROJ-123")
   ├─> Show success toast: "Uploaded to Jira as PROJ-123"
   └─> Badge becomes clickable (opens Jira issue in new tab)
```

### Security Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Security Layers                           │
├─────────────────────────────────────────────────────────────┤
│  1. Authentication                                           │
│     - User authenticates via OAuth (existing vTeam auth)    │
│     - Backend validates user token via K8s TokenReview      │
│     - RBAC checks: user must have 'get' on AgenticSession   │
├─────────────────────────────────────────────────────────────┤
│  2. Credential Isolation                                     │
│     - Jira credentials scoped to project namespace          │
│     - Secret name: {project}-jira-credentials               │
│     - Only backend SA can read secrets (not exposed to UI)  │
│     - API tokens encrypted at rest by K8s etcd encryption   │
├─────────────────────────────────────────────────────────────┤
│  3. Authorization                                            │
│     - User must have 'update' permission on AgenticSession  │
│     - Project RBAC enforced by K8s API server               │
│     - No cross-project Jira operations allowed              │
├─────────────────────────────────────────────────────────────┤
│  4. Data Validation                                          │
│     - Sanitize Markdown before conversion (XSS prevention)  │
│     - Validate Jira issue keys (regex: [A-Z]+-\d+)          │
│     - File size limits enforced (reject files >10MB)        │
│     - Rate limiting on Jira API calls (respect 300/10s)     │
├─────────────────────────────────────────────────────────────┤
│  5. Audit Trail                                              │
│     - All Jira operations logged with user ID and timestamp │
│     - K8s events track annotation updates                   │
│     - Jira maintains its own audit log of issue changes     │
└─────────────────────────────────────────────────────────────┘
```

## API Specifications

### Backend REST API

#### 1. Test Jira Connection
**Endpoint**: `POST /api/projects/:projectName/jira/test-connection`
**Auth**: Required (user token)
**Request Body**:
```json
{
  "jiraUrl": "https://your-domain.atlassian.net",
  "projectKey": "PROJ",
  "userEmail": "user@example.com",
  "apiToken": "YOUR_API_TOKEN"
}
```
**Response (200 OK)**:
```json
{
  "success": true,
  "projectName": "My Project",
  "availableIssueTypes": ["Story", "Bug", "Task", "Epic"]
}
```
**Response (401 Unauthorized)**:
```json
{
  "success": false,
  "error": "Authentication failed: Invalid credentials"
}
```

#### 2. Upload Artifact to Jira
**Endpoint**: `POST /api/projects/:projectName/sessions/:sessionName/jira/upload`
**Auth**: Required (user token)
**Request Body**:
```json
{
  "artifactPath": "artifacts/spec.md",
  "operation": "create",
  "issueDetails": {
    "summary": "Feature Specification Document",
    "issueType": "Story",
    "labels": ["feature", "spec"],
    "description": "Additional context (optional)"
  }
}
```
**Response (201 Created)**:
```json
{
  "issueKey": "PROJ-123",
  "issueUrl": "https://your-domain.atlassian.net/browse/PROJ-123",
  "artifactPath": "artifacts/spec.md",
  "syncedAt": "2025-11-11T10:30:00Z"
}
```

#### 3. Update Existing Jira Issue
**Endpoint**: `POST /api/projects/:projectName/sessions/:sessionName/jira/upload`
**Auth**: Required (user token)
**Request Body**:
```json
{
  "artifactPath": "artifacts/spec.md",
  "operation": "update",
  "issueKey": "PROJ-123"
}
```
**Response (200 OK)**:
```json
{
  "issueKey": "PROJ-123",
  "issueUrl": "https://your-domain.atlassian.net/browse/PROJ-123",
  "artifactPath": "artifacts/spec.md",
  "syncedAt": "2025-11-11T11:00:00Z",
  "updateComment": "Content updated from vTeam at 2025-11-11T11:00:00Z"
}
```

#### 4. Get Jira Links for Session
**Endpoint**: `GET /api/projects/:projectName/sessions/:sessionName/jira/links`
**Auth**: Required (user token)
**Response (200 OK)**:
```json
{
  "links": [
    {
      "artifactPath": "artifacts/spec.md",
      "issueKey": "PROJ-123",
      "issueUrl": "https://your-domain.atlassian.net/browse/PROJ-123",
      "issueSummary": "Feature Specification Document",
      "issueStatus": "In Progress",
      "syncedAt": "2025-11-11T10:30:00Z",
      "status": "active"
    }
  ]
}
```

#### 5. Delete Jira Link
**Endpoint**: `DELETE /api/projects/:projectName/sessions/:sessionName/jira/links/:artifactPath`
**Auth**: Required (user token)
**Response (204 No Content)**:
```
(Empty body)
```

#### 6. Get/Set Jira Credentials
**Endpoint**: `GET /api/projects/:projectName/jira/credentials`
**Auth**: Required (user token, project admin only)
**Response (200 OK)**:
```json
{
  "jiraUrl": "https://your-domain.atlassian.net",
  "projectKey": "PROJ",
  "userEmail": "user@example.com",
  "apiTokenSet": true
}
```

**Endpoint**: `PUT /api/projects/:projectName/jira/credentials`
**Auth**: Required (user token, project admin only)
**Request Body**:
```json
{
  "jiraUrl": "https://your-domain.atlassian.net",
  "projectKey": "PROJ",
  "userEmail": "user@example.com",
  "apiToken": "YOUR_API_TOKEN"
}
```
**Response (200 OK)**:
```json
{
  "message": "Jira credentials updated successfully"
}
```

### Jira REST API Integration

**Base URL**: `https://{jiraUrl}/rest/api/3/`
**Authentication**: HTTP Basic Auth with email and API token

#### Create Issue
**Endpoint**: `POST /rest/api/3/issue`
**Headers**:
```
Authorization: Basic {base64(email:apiToken)}
Content-Type: application/json
```
**Request Body**:
```json
{
  "fields": {
    "project": { "key": "PROJ" },
    "summary": "Feature Specification Document",
    "description": {
      "type": "doc",
      "version": 1,
      "content": [
        {
          "type": "paragraph",
          "content": [
            { "type": "text", "text": "Converted markdown content..." }
          ]
        }
      ]
    },
    "issuetype": { "name": "Story" },
    "labels": ["vteam", "auto-synced"]
  }
}
```

#### Update Issue
**Endpoint**: `PUT /rest/api/3/issue/{issueKey}`
**Request Body**: Same as create (only description field updated)

#### Get Issue
**Endpoint**: `GET /rest/api/3/issue/{issueKey}?fields=summary,status,key`
**Response**:
```json
{
  "key": "PROJ-123",
  "fields": {
    "summary": "Feature Specification Document",
    "status": { "name": "In Progress" }
  }
}
```

## Data Models

### Backend Types (Go)

```go
// components/backend/types/jira.go
package types

import "time"

// JiraCredentials represents Jira connection configuration for a project
type JiraCredentials struct {
	JiraURL    string `json:"jiraUrl" binding:"required"`
	ProjectKey string `json:"projectKey" binding:"required"`
	UserEmail  string `json:"userEmail" binding:"required"`
	APIToken   string `json:"apiToken" binding:"required"`
}

// JiraLink represents the association between an artifact file and a Jira issue
type JiraLink struct {
	ArtifactPath string    `json:"artifactPath"`
	IssueKey     string    `json:"issueKey"`
	IssueURL     string    `json:"issueUrl"`
	IssueSummary string    `json:"issueSummary,omitempty"`
	IssueStatus  string    `json:"issueStatus,omitempty"`
	SyncedAt     time.Time `json:"syncedAt"`
	Status       string    `json:"status"` // "active" | "broken"
}

// JiraUploadRequest represents a request to upload an artifact to Jira
type JiraUploadRequest struct {
	ArtifactPath string              `json:"artifactPath" binding:"required"`
	Operation    string              `json:"operation" binding:"required"` // "create" | "update"
	IssueKey     string              `json:"issueKey,omitempty"`           // Required for "update"
	IssueDetails *JiraIssueDetails   `json:"issueDetails,omitempty"`       // Required for "create"
}

// JiraIssueDetails represents details for creating a new Jira issue
type JiraIssueDetails struct {
	Summary     string   `json:"summary" binding:"required"`
	IssueType   string   `json:"issueType" binding:"required"`
	Labels      []string `json:"labels,omitempty"`
	Description string   `json:"description,omitempty"`
}

// JiraUploadResponse represents the response after uploading to Jira
type JiraUploadResponse struct {
	IssueKey      string    `json:"issueKey"`
	IssueURL      string    `json:"issueUrl"`
	ArtifactPath  string    `json:"artifactPath"`
	SyncedAt      time.Time `json:"syncedAt"`
	UpdateComment string    `json:"updateComment,omitempty"`
}

// JiraTestConnectionRequest represents a request to test Jira credentials
type JiraTestConnectionRequest struct {
	JiraURL    string `json:"jiraUrl" binding:"required"`
	ProjectKey string `json:"projectKey" binding:"required"`
	UserEmail  string `json:"userEmail" binding:"required"`
	APIToken   string `json:"apiToken" binding:"required"`
}

// JiraTestConnectionResponse represents the response from testing Jira connection
type JiraTestConnectionResponse struct {
	Success             bool     `json:"success"`
	ProjectName         string   `json:"projectName,omitempty"`
	AvailableIssueTypes []string `json:"availableIssueTypes,omitempty"`
	Error               string   `json:"error,omitempty"`
}

// JiraLinksMap represents the map of artifact paths to Jira links
// Stored as JSON in AgenticSession metadata.annotations["jira.ambient-code.io/links"]
type JiraLinksMap map[string]JiraLink
```

### Frontend Types (TypeScript)

```typescript
// components/frontend/src/types/jira.ts

export interface JiraCredentials {
  jiraUrl: string;
  projectKey: string;
  userEmail: string;
  apiToken: string;
}

export interface JiraCredentialsResponse {
  jiraUrl: string;
  projectKey: string;
  userEmail: string;
  apiTokenSet: boolean;
}

export interface JiraLink {
  artifactPath: string;
  issueKey: string;
  issueUrl: string;
  issueSummary?: string;
  issueStatus?: string;
  syncedAt: string;
  status: 'active' | 'broken';
}

export interface JiraUploadRequest {
  artifactPath: string;
  operation: 'create' | 'update';
  issueKey?: string;
  issueDetails?: JiraIssueDetails;
}

export interface JiraIssueDetails {
  summary: string;
  issueType: string;
  labels?: string[];
  description?: string;
}

export interface JiraUploadResponse {
  issueKey: string;
  issueUrl: string;
  artifactPath: string;
  syncedAt: string;
  updateComment?: string;
}

export interface JiraTestConnectionRequest {
  jiraUrl: string;
  projectKey: string;
  userEmail: string;
  apiToken: string;
}

export interface JiraTestConnectionResponse {
  success: boolean;
  projectName?: string;
  availableIssueTypes?: string[];
  error?: string;
}

export interface JiraLinksResponse {
  links: JiraLink[];
}
```

### Kubernetes Resources

#### Jira Credentials Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {project-name}-jira-credentials
  namespace: {project-name}
  labels:
    app.kubernetes.io/managed-by: vteam
    vteam.ambient-code.io/component: jira-integration
type: Opaque
stringData:
  JIRA_URL: "https://your-domain.atlassian.net"
  JIRA_PROJECT_KEY: "PROJ"
  JIRA_USER_EMAIL: "user@example.com"
  JIRA_API_TOKEN: "YOUR_API_TOKEN"
```

#### AgenticSession CRD Annotations (Extended)

```yaml
apiVersion: vteam.ambient-code/v1alpha1
kind: AgenticSession
metadata:
  name: agentic-session-1234567890
  namespace: my-project
  annotations:
    # Existing annotations...
    vteam.ambient-code/parent-session-id: "..."

    # NEW: Jira integration metadata
    jira.ambient-code.io/links: |
      {
        "artifacts/spec.md": {
          "issueKey": "PROJ-123",
          "issueUrl": "https://your-domain.atlassian.net/browse/PROJ-123",
          "issueSummary": "Feature Specification Document",
          "issueStatus": "In Progress",
          "syncedAt": "2025-11-11T10:30:00Z",
          "status": "active"
        },
        "artifacts/design.md": {
          "issueKey": "PROJ-124",
          "issueUrl": "https://your-domain.atlassian.net/browse/PROJ-124",
          "issueSummary": "System Design Document",
          "issueStatus": "To Do",
          "syncedAt": "2025-11-11T11:00:00Z",
          "status": "active"
        }
      }
```

## Technical Decisions

### 1. Jira API Version: REST API v3

**Decision**: Use Jira REST API v3 for all operations.

**Rationale**:
- v3 is the current standard for Jira Cloud (launched 2018)
- Jira Server/Data Center 8.0+ supports v3 API
- v3 uses Atlassian Document Format (ADF) for rich content
- Better structured responses with consistent field naming
- v2 API is in maintenance mode (no new features)

**Trade-offs**:
- v3 requires more complex document format (ADF instead of wiki markup)
- Some older Jira Server instances (<8.0) only support v2
- Migration path: Support v2 as fallback if v3 fails (future enhancement)

**Implementation Notes**:
- Content service will handle ADF generation from Markdown
- Fallback to plain text description if ADF conversion fails
- Version detection endpoint: `GET /rest/api/3/serverInfo`

### 2. Credential Storage: Kubernetes Secrets (Project-Scoped)

**Decision**: Store Jira credentials in project-namespaced Kubernetes Secrets.

**Rationale**:
- Aligns with existing pattern (GitHub credentials stored similarly)
- Kubernetes provides encryption at rest via etcd encryption
- Project-level isolation prevents cross-namespace access
- Automatic cleanup when project is deleted
- No additional secret management infrastructure required

**Trade-offs**:
- Single set of credentials per project (not per-user)
- Requires project admin to configure credentials
- All users in project share same Jira identity for API calls
- Alternative (rejected): User-level OAuth would require Jira App installation

**Security Considerations**:
- API tokens are base64-encoded in transit to Jira (HTTPS only)
- Backend service account has read-only access to secrets
- Frontend never receives API tokens (only confirmation of existence)
- Secrets can be rotated without code changes

### 3. Metadata Storage: AgenticSession CRD Annotations

**Decision**: Store Jira link metadata as JSON in AgenticSession annotations.

**Rationale**:
- Native Kubernetes primitive (no external database)
- Automatic lifecycle management (deleted with session)
- RBAC enforcement via K8s API server
- Supports session cloning (annotations copy with session)
- Efficient retrieval (single K8s API call)

**Trade-offs**:
- Annotation size limit: 256KB total per resource
  - Estimated capacity: ~2000 links at 128 bytes each
  - Mitigation: Warn users if approaching limit
- No complex querying (can't search by issue key across sessions)
  - Mitigation: Session-level queries are sufficient for UI
- JSON parsing required on every read
  - Mitigation: Parse once and cache in memory

**Alternative Considered**: Custom Resource Definition (JiraLink CRD)
- Rejected: Adds complexity, requires CRD installation, overkill for simple mapping

### 4. Markdown Conversion: Atlassian Document Format (ADF)

**Decision**: Convert Markdown to ADF (Atlassian Document Format) for Jira issue descriptions.

**Rationale**:
- ADF is Jira Cloud's native rich content format (v3 API)
- Preserves formatting (headers, lists, code blocks, links)
- Better readability in Jira UI compared to plain text
- Supports Jira-specific features (mentions, tables, emojis)

**Conversion Strategy**:
```
Markdown Element     →  ADF Node Type
───────────────────────────────────────
# Header            →  heading (level 1)
## Subheader        →  heading (level 2)
**bold**            →  text (marks: [strong])
*italic*            →  text (marks: [em])
[link](url)         →  text (marks: [link])
- list item         →  bulletList > listItem
1. ordered item     →  orderedList > listItem
```code```         →  codeBlock
> blockquote        →  blockquote
```

**Fallback Strategy**:
1. Attempt ADF conversion
2. If conversion fails, fallback to Jira Wiki Markup
3. If Wiki Markup fails, use plain text
4. Log conversion errors for debugging

**Library**: Use existing Go Markdown parser (e.g., `goldmark`) + custom ADF renderer

### 5. UI Integration: Context Menu + Badges

**Decision**: Integrate Jira actions via right-click context menu on artifact files + visual badges.

**Rationale**:
- Familiar pattern (used for file operations in vTeam)
- Non-intrusive (doesn't clutter primary UI)
- Discoverable (right-click is standard interaction)
- Badges provide at-a-glance sync status without extra clicks

**UI Components**:
```
Artifacts Panel
├── File Row
│   ├── File Icon
│   ├── File Name
│   ├── Jira Badge (if synced)  →  Shows "PROJ-123"
│   │   └── Tooltip: "Linked to: Feature Spec (In Progress)"
│   └── Context Menu (right-click)
│       ├── Download
│       ├── Rename
│       ├── Delete
│       ├── ───────────
│       ├── Upload to Jira
│       │   ├── Create New Issue
│       │   └── Update Existing Issue
│       ├── View in Jira (if synced)
│       ├── Copy Jira Link (if synced)
│       └── Unlink from Jira (if synced)
```

**Badge States**:
- Green badge: Successfully synced, issue is open
- Gray badge: Issue is closed or resolved
- Red badge: Link is broken (issue deleted or inaccessible)
- No badge: Not synced to Jira

### 6. Rate Limiting: Client-Side Throttling

**Decision**: Implement client-side rate limiting for Jira API calls (300 requests/10 seconds).

**Rationale**:
- Jira Cloud enforces strict rate limits per user
- Prevents exhausting rate limit quota
- Improves reliability (avoids 429 errors)
- Backend can queue requests and retry with exponential backoff

**Implementation**:
```go
// components/backend/jira/client.go
type RateLimiter struct {
    requests []time.Time
    maxRequests int
    window time.Duration
    mu sync.Mutex
}

func (r *RateLimiter) Wait() {
    r.mu.Lock()
    defer r.mu.Unlock()

    now := time.Now()
    // Remove requests outside the time window
    validRequests := []time.Time{}
    for _, req := range r.requests {
        if now.Sub(req) < r.window {
            validRequests = append(validRequests, req)
        }
    }
    r.requests = validRequests

    if len(r.requests) >= r.maxRequests {
        // Wait until oldest request expires
        sleepDuration := r.window - now.Sub(r.requests[0])
        time.Sleep(sleepDuration)
    }

    r.requests = append(r.requests, now)
}
```

**Alternative Considered**: Server-side rate limiting (Redis/Memcached)
- Rejected: Adds infrastructure dependency, overkill for single-project scope

### 7. Error Handling: Graceful Degradation

**Decision**: Implement graceful degradation for Jira integration failures.

**User-Facing Error Messages**:
```
Error Type                    →  User Message
────────────────────────────────────────────────────────────────
401 Unauthorized              →  "Authentication failed. Check your Jira credentials in Project Settings."
403 Forbidden                 →  "You don't have permission to create issues in project PROJ. Contact your Jira admin."
404 Not Found (Issue)         →  "Jira issue PROJ-123 not found. It may have been deleted. Would you like to unlink this artifact?"
404 Not Found (Project)       →  "Jira project PROJ not found. Verify the project key in settings."
429 Rate Limit Exceeded       →  "Jira rate limit exceeded. Please try again in a few seconds."
500 Internal Server Error     →  "Jira server error. Please try again later or contact support."
Network Timeout               →  "Connection to Jira timed out. Check your network and try again."
Markdown Conversion Failed    →  "Warning: Formatting may be lost. Uploading as plain text."
File Size Exceeded (>10MB)    →  "File too large to upload. Maximum size is 10MB."
```

**Backend Logging**:
- Log all Jira API errors with full context (user, session, artifact path)
- Include response bodies for debugging (sanitized to remove credentials)
- Track success/failure metrics for monitoring

## Implementation Phases

### Phase 1: Core Infrastructure (P1 - MVP)

**Goal**: Enable basic upload and view functionality.

**Backend Tasks**:
1. Restore and refactor `components/backend/jira/integration.go`
   - Remove RFE-specific code
   - Adapt for session-based architecture
2. Create `components/backend/jira/client.go`
   - Jira REST API v3 client wrapper
   - HTTP Basic Auth implementation
   - Rate limiting logic
3. Create `components/backend/jira/markdown.go`
   - Markdown parser (using `goldmark`)
   - ADF (Atlassian Document Format) generator
   - Fallback to plain text
4. Create `components/backend/handlers/jira.go`
   - `UploadArtifactToJira` handler
   - `GetJiraLinks` handler
   - Credential retrieval from Kubernetes Secrets
5. Update `components/backend/routes.go`
   - Add Jira API routes
   - Apply authentication middleware
6. Create `components/backend/types/jira.go`
   - Define request/response types
   - Validation logic

**Frontend Tasks**:
1. Create `components/frontend/src/services/api/jira.ts`
   - API client methods
   - Type-safe request/response handling
2. Create `components/frontend/src/services/queries/use-jira.ts`
   - TanStack Query hooks
   - Cache invalidation strategies
3. Modify `components/frontend/src/components/workspace-sections/artifacts-panel.tsx`
   - Add context menu items ("Upload to Jira")
   - Add Jira badges to file rows
   - Add upload dialog component
4. Create `components/frontend/src/types/jira.ts`
   - Frontend type definitions
   - Match backend types

**Testing**:
- Unit tests for Markdown converter (50+ test cases covering edge cases)
- Integration test: Upload artifact to test Jira instance
- E2E test: Full upload flow from UI to Jira and back

**Acceptance Criteria**:
- User can upload a Markdown file to Jira as a new issue
- Jira badge displays on synced files
- Clicking badge opens Jira issue in new tab
- Markdown formatting preserved in Jira (headers, lists, code blocks)

### Phase 2: Credential Management (P2)

**Goal**: Enable users to configure and test Jira credentials.

**Backend Tasks**:
1. Create `components/backend/jira/validation.go`
   - Credential validation logic
   - Test connection function
2. Add handlers to `components/backend/handlers/jira.go`
   - `TestJiraConnection` handler
   - `GetJiraCredentials` handler
   - `UpdateJiraCredentials` handler
3. Create Kubernetes Secret on credential save
   - Secret name: `{project}-jira-credentials`
   - Encrypted fields: JIRA_API_TOKEN

**Frontend Tasks**:
1. Modify `components/frontend/src/components/workspace-sections/settings-section.tsx`
   - Add "Test Connection" button
   - Display connection status (success/failure)
   - Show error messages with actionable guidance
2. Add validation:
   - URL format validation (https:// required)
   - Project key format (uppercase alphanumeric)
   - Email format validation

**Testing**:
- Unit test: Credential validation logic
- Integration test: Test connection with valid/invalid credentials
- E2E test: Save credentials, test connection, upload artifact

**Acceptance Criteria**:
- User can configure Jira credentials in Project Settings
- "Test Connection" validates credentials and shows project info
- Credentials persist across sessions
- API token is masked in UI

### Phase 3: Update and Link Management (P2)

**Goal**: Enable updating existing issues and managing links.

**Backend Tasks**:
1. Extend `UploadArtifactToJira` handler to support "update" operation
2. Add `DeleteJiraLink` handler
   - Remove annotation from AgenticSession
3. Add issue existence check before update
   - Handle 404 (issue deleted) gracefully

**Frontend Tasks**:
1. Add context menu items:
   - "Update Existing Issue"
   - "Unlink from Jira"
2. Add confirmation dialog for unlink action
3. Handle broken links:
   - Display red badge for 404 errors
   - Offer to unlink or re-link

**Testing**:
- Unit test: Update operation logic
- Integration test: Update existing issue
- E2E test: Update issue, verify content changed in Jira

**Acceptance Criteria**:
- User can update an existing Jira issue with modified artifact content
- User can unlink an artifact from Jira
- Broken links are visually distinguished and actionable

### Phase 4: Polish and Optimization (P3)

**Goal**: Improve user experience and performance.

**Backend Tasks**:
1. Implement batch upload support
   - Queue multiple files
   - Progress tracking
2. Add issue metadata caching
   - Cache issue summary/status for 5 minutes
   - Reduce API calls for badge tooltips
3. Implement retry logic with exponential backoff
   - Retry on network errors (max 3 attempts)

**Frontend Tasks**:
1. Add multi-select UI for bulk upload
   - Checkboxes on file rows
   - "Upload Selected to Jira" button
2. Add progress indicator for batch operations
   - Show "3 of 5 completed"
   - Display individual success/failure states
3. Optimize badge rendering
   - Lazy load issue status (on hover)
   - Cache badge data in component state

**Testing**:
- Performance test: Upload 50 files in batch (should complete <2 minutes)
- UI test: Verify progress indicator updates correctly
- E2E test: Bulk upload with mixed success/failure

**Acceptance Criteria**:
- User can select and upload multiple artifacts in one operation
- Progress indicator shows real-time status
- Failed uploads can be retried without re-uploading successful ones

## Migration and Rollout Strategy

### Pre-Deployment Checklist

1. **Jira Test Instance Setup**
   - Create test Jira Cloud project
   - Generate API token for testing
   - Configure test credentials in dev environment

2. **Documentation**
   - User guide: How to configure Jira integration
   - Admin guide: Kubernetes Secret setup
   - API documentation: OpenAPI spec for Jira endpoints

3. **Monitoring Setup**
   - Prometheus metrics for Jira API calls (success/failure rates)
   - Grafana dashboard for rate limit tracking
   - Alert rules for credential expiration

### Deployment Plan

**Phase 1: Alpha Testing (Internal Team)**
- Deploy to staging environment
- Test with internal vTeam project
- Validate all P1 user stories
- Fix critical bugs

**Phase 2: Beta Testing (Select Users)**
- Deploy to production with feature flag
- Enable for 2-3 pilot projects
- Collect feedback on UX and reliability
- Monitor error rates and performance

**Phase 3: General Availability**
- Remove feature flag
- Enable for all projects
- Publish user documentation
- Announce via release notes

### Rollback Plan

If critical issues are discovered:
1. Disable Jira API routes via feature flag
2. Jira badges remain visible but actions are disabled
3. No data loss (links stored in annotations persist)
4. Fix issues in staging environment
5. Re-deploy and re-enable

### Backward Compatibility

- **Session Metadata**: Annotations are additive (existing sessions unaffected)
- **API Routes**: New routes don't conflict with existing endpoints
- **Frontend**: Jira UI components are opt-in (only shown if credentials configured)

## Performance Considerations

### Expected Load

- **Concurrent Users**: 50 users across all projects
- **Jira API Calls**: ~500 requests/hour peak (10 uploads/hour × 50 users)
- **Rate Limit Budget**: 300 requests/10 seconds = 1800 requests/minute
- **Headroom**: 96% available capacity during peak load

### Optimization Strategies

1. **Caching**:
   - Cache issue metadata (summary, status) for 5 minutes
   - Reduce badge tooltip API calls by 90%

2. **Batching**:
   - Batch issue metadata fetches (single API call for multiple issues)
   - Use Jira JQL search to fetch all issues in one request

3. **Lazy Loading**:
   - Load issue status only when badge is hovered
   - Defer non-critical metadata until user interaction

4. **Connection Pooling**:
   - Reuse HTTP connections to Jira API
   - Configure keep-alive timeouts (5 minutes)

### Performance Targets (from Technical Context)

- ✓ Jira API calls: <5s p95 latency (Jira Cloud typically <500ms)
- ✓ UI responsiveness: <200ms for badge rendering (in-memory JSON parse)
- ✓ Markdown conversion: <1s for 50KB files (benchmarked at <100ms)

## Risk Assessment

### High Risk

**Risk**: Jira API rate limits exhausted during bulk uploads
**Mitigation**: Client-side rate limiting + queue with backoff
**Contingency**: Display "Slow down" message, pause uploads for 10 seconds

**Risk**: Jira credentials leaked via logs or error messages
**Mitigation**: Sanitize all log outputs, never log API tokens
**Contingency**: Rotate compromised credentials, audit access logs

### Medium Risk

**Risk**: Markdown conversion failures for complex documents
**Mitigation**: Comprehensive test suite (50+ edge cases), fallback to plain text
**Contingency**: Manual cleanup in Jira, user notification with diff view

**Risk**: Annotation size limit exceeded (256KB) for projects with 1000+ links
**Mitigation**: Warn at 80% capacity, suggest archiving old sessions
**Contingency**: Implement pagination or external storage for links (future)

### Low Risk

**Risk**: Jira instance becomes unavailable during upload
**Mitigation**: Retry logic with exponential backoff (3 attempts)
**Contingency**: Display error message, allow retry later

## Success Metrics

### Phase 1 (MVP) Success Criteria

- ✓ 90% of uploads complete successfully (<10% failure rate)
- ✓ Jira badges render within 200ms of file list load
- ✓ Markdown formatting preserved for 95% of common elements
- ✓ Zero credential leakage incidents
- ✓ User can upload artifact and open in Jira within 15 seconds

### Phase 2 (Full Release) Success Criteria

- ✓ 5+ projects actively using Jira integration
- ✓ 100+ artifacts synced to Jira per week
- ✓ User satisfaction score: 4.5/5 (post-release survey)
- ✓ Support ticket rate: <5 tickets/month related to Jira integration
- ✓ 80% of users who configure Jira upload at least 1 artifact within first session

### Monitoring Dashboards

**Jira Integration Health Dashboard**:
- API call success rate (target: >95%)
- API latency p50/p95/p99 (target: <500ms p95)
- Rate limit utilization (target: <50% of quota)
- Credential test success rate (target: >90%)
- Markdown conversion success rate (target: >95%)

**User Adoption Metrics**:
- Projects with Jira configured (count)
- Active uploads per day (count)
- Unique users uploading per week (count)
- Average uploads per active user (count)

## Open Questions and Future Enhancements

### Open Questions (Requires Clarification)

1. **Jira API Version Support**: Should we support Jira Server/Data Center v2 API as fallback?
   - **Decision Needed**: Minimum supported Jira version
   - **Impact**: Affects ADF vs Wiki Markup conversion strategy

2. **File Size Limit**: What is the maximum file size for Jira uploads?
   - **Recommendation**: 10MB hard limit (align with Jira Cloud limits)
   - **Decision Needed**: Confirm with stakeholders

3. **Credential Scope**: Should credentials be per-user or per-project?
   - **Current Design**: Per-project (shared)
   - **Alternative**: Per-user OAuth (requires Jira App installation)
   - **Decision Needed**: Validate with security team

### Future Enhancements (Post-MVP)

1. **Bidirectional Sync**:
   - Pull Jira comments into session workspace
   - Sync issue status changes to session annotations
   - Webhook listener for real-time updates

2. **Advanced Markdown Features**:
   - Support for images (upload as attachments)
   - Support for tables (convert to Jira table format)
   - Support for Mermaid diagrams (render as images)

3. **Bulk Operations**:
   - Bulk unlink artifacts
   - Bulk update (re-sync all modified artifacts)
   - Archive synced artifacts to Jira when session closes

4. **Jira Issue Templates**:
   - Predefined issue templates (Story, Bug, Task)
   - Custom field mappings (e.g., Story Points, Sprint)
   - Template library in Project Settings

5. **Search and Filtering**:
   - Search artifacts by Jira issue key
   - Filter by sync status (synced, not synced, broken)
   - Sort by last sync time

6. **Audit Trail**:
   - Display sync history for each artifact
   - Show who synced and when
   - Link to Jira issue history

## Appendix: Code Examples

### Example: Jira Client Implementation

```go
// components/backend/jira/client.go
package jira

import (
    "bytes"
    "encoding/base64"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
)

type Client struct {
    BaseURL    string
    Email      string
    APIToken   string
    HTTPClient *http.Client
    RateLimiter *RateLimiter
}

func NewClient(baseURL, email, apiToken string) *Client {
    return &Client{
        BaseURL:  baseURL,
        Email:    email,
        APIToken: apiToken,
        HTTPClient: &http.Client{
            Timeout: 30 * time.Second,
        },
        RateLimiter: NewRateLimiter(300, 10*time.Second),
    }
}

func (c *Client) CreateIssue(req CreateIssueRequest) (*IssueResponse, error) {
    c.RateLimiter.Wait() // Enforce rate limiting

    url := fmt.Sprintf("%s/rest/api/3/issue", c.BaseURL)
    body, err := json.Marshal(req)
    if err != nil {
        return nil, fmt.Errorf("marshal request: %w", err)
    }

    httpReq, err := http.NewRequest("POST", url, bytes.NewReader(body))
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }

    // Set authentication header
    auth := base64.StdEncoding.EncodeToString([]byte(c.Email + ":" + c.APIToken))
    httpReq.Header.Set("Authorization", "Basic "+auth)
    httpReq.Header.Set("Content-Type", "application/json")

    resp, err := c.HTTPClient.Do(httpReq)
    if err != nil {
        return nil, fmt.Errorf("send request: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusCreated {
        bodyBytes, _ := io.ReadAll(resp.Body)
        return nil, fmt.Errorf("jira API error %d: %s", resp.StatusCode, string(bodyBytes))
    }

    var issueResp IssueResponse
    if err := json.NewDecoder(resp.Body).Decode(&issueResp); err != nil {
        return nil, fmt.Errorf("decode response: %w", err)
    }

    return &issueResp, nil
}
```

### Example: Frontend Upload Dialog

```tsx
// components/frontend/src/components/jira-upload-dialog.tsx
import { useState } from 'react';
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { useUploadToJira } from '@/services/queries/use-jira';
import { successToast, errorToast } from '@/hooks/use-toast';

interface JiraUploadDialogProps {
  open: boolean;
  onClose: () => void;
  artifactPath: string;
  projectName: string;
  sessionName: string;
}

export function JiraUploadDialog({
  open,
  onClose,
  artifactPath,
  projectName,
  sessionName,
}: JiraUploadDialogProps) {
  const [summary, setSummary] = useState(artifactPath.split('/').pop() || '');
  const [issueType, setIssueType] = useState('Story');
  const [labels, setLabels] = useState('feature,spec');

  const uploadMutation = useUploadToJira();

  const handleUpload = () => {
    uploadMutation.mutate(
      {
        projectName,
        sessionName,
        request: {
          artifactPath,
          operation: 'create',
          issueDetails: {
            summary,
            issueType,
            labels: labels.split(',').map(l => l.trim()).filter(Boolean),
          },
        },
      },
      {
        onSuccess: (data) => {
          successToast(`Created ${data.issueKey} in Jira`);
          onClose();
        },
        onError: (error) => {
          errorToast(error.message);
        },
      }
    );
  };

  return (
    <Dialog open={open} onOpenChange={onClose}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Upload to Jira</DialogTitle>
        </DialogHeader>
        <div className="space-y-4">
          <div>
            <Label htmlFor="summary">Summary</Label>
            <Input
              id="summary"
              value={summary}
              onChange={(e) => setSummary(e.target.value)}
              placeholder="Feature specification document"
            />
          </div>
          <div>
            <Label htmlFor="issueType">Issue Type</Label>
            <select
              id="issueType"
              value={issueType}
              onChange={(e) => setIssueType(e.target.value)}
              className="w-full"
            >
              <option value="Story">Story</option>
              <option value="Task">Task</option>
              <option value="Bug">Bug</option>
              <option value="Epic">Epic</option>
            </select>
          </div>
          <div>
            <Label htmlFor="labels">Labels (comma-separated)</Label>
            <Input
              id="labels"
              value={labels}
              onChange={(e) => setLabels(e.target.value)}
              placeholder="feature, spec, auto-synced"
            />
          </div>
          <div className="flex justify-end gap-2">
            <Button variant="outline" onClick={onClose}>
              Cancel
            </Button>
            <Button onClick={handleUpload} disabled={uploadMutation.isPending}>
              {uploadMutation.isPending ? 'Uploading...' : 'Upload to Jira'}
            </Button>
          </div>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

---

This implementation plan provides a comprehensive roadmap for restoring Jira integration to the vTeam platform. The design aligns with established patterns, prioritizes security and scalability, and delivers user value through incremental phases. The architectural approach of using Kubernetes Secrets for credentials and CRD annotations for metadata ensures the integration remains maintainable and secure while minimizing infrastructure complexity.
