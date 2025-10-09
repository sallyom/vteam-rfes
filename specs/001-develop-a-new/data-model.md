# Data Model: Jira-Session Artifact Integration

**Date**: 2025-10-09
**Status**: Draft

## Overview
Data structures and relationships for linking AgenticSession artifacts to Jira issues.

---

## 1. Core Entities

### 1.1 JiraLink
**Purpose**: Represents a link between a session artifact and a Jira issue

**Storage**: AgenticSession CR metadata annotations (JSON-serialized array)

**Schema**:
```typescript
interface JiraLink {
  path: string          // Artifact file path (e.g., "transcript.txt")
  jiraKey: string       // Jira issue key (e.g., "PROJ-123")
  timestamp: string     // ISO 8601 timestamp (e.g., "2025-10-09T10:00:00Z")
  status: string        // "success" | "failed"
  error?: string        // Error message if status is "failed"
}
```

**Kubernetes Annotation Format**:
```yaml
metadata:
  annotations:
    vteam.ambient-code/jiraLinks: '[{"path":"transcript.txt","jiraKey":"PROJ-123","timestamp":"2025-10-09T10:00:00Z","status":"success"}]'
```

**Validation Rules**:
- `path` must not be empty, max 255 characters
- `jiraKey` must match pattern `^[A-Z][A-Z0-9]+-[0-9]+$`
- `timestamp` must be valid ISO 8601 format
- `status` must be "success" or "failed"
- `error` max 1000 characters

**Relationships**:
- One AgenticSession → Many JiraLinks
- One JiraLink → One artifact file
- One Jira issue can have multiple JiraLinks from different artifacts/sessions

---

### 1.2 SessionArtifact
**Purpose**: Represents a file generated during an agentic session

**Storage**: Filesystem (stateDir in AgenticSession status)

**Schema**:
```typescript
interface SessionArtifact {
  path: string          // Relative path within stateDir
  size: number          // File size in bytes
  mimeType: string      // MIME type (e.g., "text/plain", "application/json")
  lastModified: string  // ISO 8601 timestamp
}
```

**Common Artifact Types**:
| Path | MIME Type | Description |
|------|-----------|-------------|
| `transcript.txt` | text/plain | Full conversation history |
| `result_summary.txt` | text/plain | Final result summary |
| `tool_outputs/*.log` | text/plain | Tool execution logs |
| `state_snapshot.json` | application/json | Session state dump |
| `errors.log` | text/plain | Error details (failed sessions) |

**Validation Rules**:
- `path` must not contain `..` (no path traversal)
- `size` must be ≤ 10,485,760 bytes (10MB Jira limit)
- `mimeType` must be valid MIME type

**Relationships**:
- One AgenticSession → Many SessionArtifacts (via stateDir)
- One SessionArtifact → Zero or more JiraLinks

---

### 1.3 JiraConfiguration
**Purpose**: Project-scoped Jira connection settings

**Storage**: Kubernetes Secret (runner secrets)

**Schema**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ambient-runner-secrets  # Or custom name from ProjectSettings
  namespace: <project-namespace>
type: Opaque
stringData:
  JIRA_URL: "https://company.atlassian.net"  # Jira instance URL
  JIRA_PROJECT: "VTEAM"                      # Default project key
  JIRA_API_TOKEN: "ATATT3xFf..."             # API token
```

**Validation Rules**:
- `JIRA_URL` must be valid HTTPS URL
- `JIRA_PROJECT` must match `^[A-Z][A-Z0-9]+$`
- `JIRA_API_TOKEN` must not be empty, typically starts with "ATATT" for Cloud

**Relationships**:
- One Kubernetes namespace → One JiraConfiguration (via runner secrets)
- JiraConfiguration applies to all AgenticSessions in namespace

---

### 1.4 PushOperation
**Purpose**: Represents a user-initiated push of artifacts to Jira

**Storage**: Transient (request/response only, history in JiraLinks)

**Schema**:
```typescript
interface PushRequest {
  issueKey: string        // Target Jira issue key
  artifacts: string[]     // Array of artifact paths to push
}

interface PushResponse {
  success: boolean
  jiraKey: string
  attachments: string[]   // Successfully uploaded artifacts
  commentId?: string      // Jira comment ID if created
  errors?: {              // Per-artifact errors for partial failures
    path: string
    error: string
  }[]
}
```

**State Transitions**:
```
1. User requests push → Validate issue → Load artifacts
2. Upload artifacts → Track success/failure per file
3. Create comment → Update AgenticSession annotations
4. Return response with success/error details
```

---

## 2. Extended AgenticSession Schema

### Current Schema (agenticsessions-crd.yaml)
```yaml
spec:
  repos: [...]
  prompt: string
  llmSettings: {...}
  timeout: integer
  interactive: boolean

status:
  phase: string  # Pending|Creating|Running|Completed|Failed|Stopped
  message: string
  stateDir: string  # <-- Artifact source
  result: string
  total_cost_usd: number
  num_turns: integer
  # ... other result fields
```

### Extended with Jira Metadata (annotations)
```yaml
metadata:
  annotations:
    vteam.ambient-code/jiraLinks: '[{...}]'  # JiraLink array (JSON)
    vteam.ambient-code/lastJiraPush: "2025-10-09T10:00:00Z"  # Latest push timestamp
```

**Note**: No CRD schema changes required. All Jira metadata stored in annotations.

---

## 3. API Request/Response Models

### 3.1 List Artifacts
```typescript
// GET /api/projects/:projectName/sessions/:sessionName/artifacts
// Response:
{
  artifacts: SessionArtifact[]
}
```

### 3.2 Validate Jira Issue
```typescript
// POST /api/projects/:projectName/sessions/:sessionName/jira/validate
// Request:
{
  issueKey: string
}
// Response:
{
  valid: boolean
  issue?: {
    key: string
    summary: string
    status: string
    project: string
  }
  error?: string
}
```

### 3.3 Push to Jira
```typescript
// POST /api/projects/:projectName/sessions/:sessionName/jira
// Request: PushRequest
// Response: PushResponse
```

### 3.4 Get Jira Links
```typescript
// GET /api/projects/:projectName/sessions/:sessionName/jira
// Response:
{
  links: JiraLink[]
}
```

---

## 4. Jira REST API Models

### 4.1 Jira Issue (GET /rest/api/2/issue/{issueKey})
```json
{
  "key": "PROJ-123",
  "fields": {
    "summary": "Issue title",
    "status": {
      "name": "In Progress"
    },
    "project": {
      "key": "PROJ"
    }
  }
}
```

### 4.2 Jira Attachment Upload (POST /rest/api/2/issue/{issueKey}/attachments)
**Request**: multipart/form-data with file field
**Response**:
```json
[
  {
    "id": "10001",
    "filename": "transcript.txt",
    "size": 12345,
    "mimeType": "text/plain"
  }
]
```

### 4.3 Jira Comment (POST /rest/api/2/issue/{issueKey}/comment)
**Request**:
```json
{
  "body": "🤖 vTeam AgenticSession: session-xyz..."
}
```
**Response**:
```json
{
  "id": "10002",
  "body": "🤖 vTeam AgenticSession: session-xyz...",
  "created": "2025-10-09T10:00:00.000+0000"
}
```

---

## 5. State Diagrams

### 5.1 Push Operation Flow
```
[User Clicks Push Button]
    ↓
[Frontend: Open Push Dialog]
    ↓
[Load Available Artifacts] ← GET /artifacts
    ↓
[User Enters Issue Key]
    ↓
[Validate Issue] ← POST /jira/validate
    ↓
[User Selects Artifacts & Submits]
    ↓
[Backend: Load Jira Config from Secret]
    ↓
[Backend: Read Artifact Files from stateDir]
    ↓
[Backend: Upload Each Artifact to Jira] ← POST /rest/api/2/issue/{key}/attachments
    ↓
[Backend: Create Comment with Session Metadata] ← POST /rest/api/2/issue/{key}/comment
    ↓
[Backend: Update AgenticSession Annotations]
    ↓
[Frontend: Show Success/Errors]
    ↓
[Display Jira Badge on Session]
```

### 5.2 Jira Link Lifecycle
```
[No Link] → [User Initiates Push] → [Link Created (status=success)] → [Displayed in UI]
                ↓
          [Push Fails] → [Link Created (status=failed)] → [Retry Available]
                ↓
          [Retry Success] → [Link Updated (status=success)]
```

---

## 6. Error Handling Models

### Error Response Structure
```typescript
interface ErrorResponse {
  error: string        // Human-readable error message
  code: string         // Error code (e.g., "JIRA_AUTH_FAILED")
  details?: string     // Additional context
  retryable: boolean   // Whether operation can be retried
}
```

### Error Codes
| Code | HTTP Status | Retryable | Description |
|------|-------------|-----------|-------------|
| `JIRA_CONFIG_MISSING` | 400 | false | Runner secret not configured |
| `JIRA_INVALID_ISSUE_KEY` | 400 | false | Issue key format invalid |
| `JIRA_ISSUE_NOT_FOUND` | 404 | false | Issue doesn't exist |
| `JIRA_AUTH_FAILED` | 401 | false | Invalid API token |
| `JIRA_PERMISSION_DENIED` | 403 | false | User lacks Jira permissions |
| `JIRA_NETWORK_ERROR` | 502 | true | Network timeout/connection failed |
| `JIRA_RATE_LIMIT` | 429 | true | API rate limit exceeded |
| `ARTIFACT_TOO_LARGE` | 400 | false | File exceeds 10MB |
| `ARTIFACT_NOT_FOUND` | 404 | false | Artifact missing from stateDir |

---

## 7. Indexing & Queries

### Kubernetes Label Selectors
Query sessions with Jira links (via annotation):
```bash
kubectl get agenticsessions -n <namespace> \
  -o json | jq '.items[] | select(.metadata.annotations["vteam.ambient-code/jiraLinks"] != null)'
```

### Backend Query Patterns
```go
// Get all Jira links for a session
func getJiraLinks(cr *unstructured.Unstructured) ([]JiraLink, error) {
  ann := cr.GetAnnotations()
  if linksJSON, ok := ann["vteam.ambient-code/jiraLinks"]; ok {
    var links []JiraLink
    if err := json.Unmarshal([]byte(linksJSON), &links); err != nil {
      return nil, err
    }
    return links, nil
  }
  return []JiraLink{}, nil
}

// Add a new Jira link
func addJiraLink(cr *unstructured.Unstructured, link JiraLink) error {
  links, _ := getJiraLinks(cr)
  links = append(links, link)
  linksJSON, _ := json.Marshal(links)

  ann := cr.GetAnnotations()
  if ann == nil {
    ann = make(map[string]string)
  }
  ann["vteam.ambient-code/jiraLinks"] = string(linksJSON)
  ann["vteam.ambient-code/lastJiraPush"] = time.Now().Format(time.RFC3339)
  cr.SetAnnotations(ann)
  return nil
}
```

---

## 8. Data Validation

### Input Validation (Backend)
```go
// Validate issue key format
func validateIssueKey(key string) error {
  re := regexp.MustCompile(`^[A-Z][A-Z0-9]+-[0-9]+$`)
  if !re.MatchString(key) {
    return fmt.Errorf("invalid issue key format: %s", key)
  }
  return nil
}

// Validate artifact size
func validateArtifactSize(size int64) error {
  const maxSize = 10 * 1024 * 1024 // 10MB
  if size > maxSize {
    return fmt.Errorf("artifact too large: %d bytes (max %d)", size, maxSize)
  }
  return nil
}
```

### Schema Validation (Frontend)
```typescript
const issueKeyRegex = /^[A-Z][A-Z0-9]+-[0-9]+$/

function validateIssueKey(key: string): string | null {
  if (!key.trim()) return "Issue key is required"
  if (!issueKeyRegex.test(key)) return "Invalid format (expected: PROJ-123)"
  return null
}

function validateArtifactSelection(artifacts: string[]): string | null {
  if (artifacts.length === 0) return "Select at least one artifact"
  return null
}
```

---

## 9. Migration & Backwards Compatibility

### Existing Sessions
- **No migration required**: Existing AgenticSessions without Jira links continue to work
- **Annotations optional**: Missing `jiraLinks` annotation is treated as empty array

### Version Compatibility
- **CRD unchanged**: Only annotations added, no schema version bump
- **API versioned**: New endpoints under `/jira` prefix, no changes to existing APIs
- **Frontend graceful**: Show "Push to Jira" only if backend supports it (feature detection)

---

## Summary

This data model:
1. **Reuses existing structures**: AgenticSession CR, runner secrets pattern
2. **Minimizes schema changes**: All Jira metadata in annotations
3. **Supports partial failures**: Per-artifact error tracking
4. **Enables audit trail**: Timestamps and status for all links
5. **Maintains consistency**: Follows RFEWorkflow JiraLink pattern

**Key Design Principles**:
- **Kubernetes-native**: Annotations, secrets, namespaces
- **API-first**: Clear request/response contracts
- **Error-aware**: Explicit error codes and retry semantics
- **Performance-conscious**: Streaming uploads, parallel processing
