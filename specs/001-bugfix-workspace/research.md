# Technical Research: BugFix Workspace Implementation

**Date**: 2025-10-31
**Feature**: BugFix Workspace Type for vTeam Platform

---

## Executive Summary

This research document consolidates technical decisions and patterns for implementing the BugFix Workspace Type within the existing vTeam platform. The platform already provides robust infrastructure for GitHub/Jira integration, workspace orchestration, and Git operations. The BugFix Workspace will follow established patterns from the RFE (Request For Enhancement) Workspace implementation.

**Key Finding**: All required technical infrastructure exists in the vTeam codebase. Implementation will primarily involve creating bug-specific handlers, CRDs, and UI components following proven patterns.

---

## 1. GitHub API Integration

### Decision
**Use existing GitHub App authentication infrastructure with direct REST API calls via `net/http`**

### Rationale
- vTeam already implements GitHub App-based authentication with JWT token generation and installation token caching
- No third-party GitHub SDK is used; all operations use direct HTTP requests with proper headers
- Token management infrastructure supports both GitHub Cloud and Enterprise
- Fallback to Personal Access Tokens from Kubernetes Secrets already implemented

### Existing Patterns to Reuse

**Authentication** (`/components/backend/github/token.go`):
- `TokenManager.MintInstallationTokenForHost()` - Get short-lived tokens from GitHub App
- Token caching with 3-minute buffer before expiry
- `GetGitHubToken()` - Unified function that tries GitHub App first, falls back to project secrets

**API Request Helper** (`/components/backend/handlers/github_auth.go:66-88`):
```go
func doGitHubRequest(ctx context.Context, method, url, authHeader, accept string, body io.Reader) (*http.Response, error)
```
- Standard headers: `Authorization`, `Accept`, `X-GitHub-Api-Version: 2022-11-28`, `User-Agent: vTeam-Backend`
- 15-second timeout
- Context-aware cancellation

**Error Handling**:
- Rate limit detection: Check `X-RateLimit-Remaining`, return reset time from `X-RateLimit-Reset` header
- Status code handling: 404 (not found), 401/403 (auth failure), 410 (deleted), 429 (rate limit)
- Pass through full GitHub API error messages for debugging

### Implementation Requirements for BugFix Workspace

Create `/components/backend/github/issues.go` with functions:

1. **CreateIssue**: POST `/repos/{owner}/{repo}/issues`
   - Input: owner, repo, title, body, token
   - Output: issue number, issue URL
   - Headers: `Authorization: Bearer {token}`, `Accept: application/vnd.github+json`

2. **UpdateIssue**: PATCH `/repos/{owner}/{repo}/issues/{number}`
   - Input: owner, repo, issue number, updated title/body, token
   - Output: success/error

3. **AddComment**: POST `/repos/{owner}/{repo}/issues/{number}/comments`
   - Input: owner, repo, issue number, comment body, token
   - Used to post technical analysis, resolution plan, and implementation summary

4. **ValidateIssueURL**: GET `/repos/{owner}/{repo}/issues/{number}`
   - Input: GitHub Issue URL, token
   - Output: owner, repo, issue number (parsed from URL)
   - Error cases: 404 (not found), 403 (no access), 410 (deleted)

5. **ParseGitHubIssueURL**: URL parsing utility
   - Input: `https://github.com/owner/repo/issues/123`
   - Output: owner, repo, issue number
   - Validation: Ensure URL format is correct before API call

### Alternatives Considered
- **go-github SDK**: Rejected due to added dependency, slower performance, larger binary size
- **GitHub GraphQL API**: Rejected; REST API v3 is simpler and sufficient for issue operations

---

## 2. Jira API Integration

### Decision
**Use existing Jira REST API v2 integration with native `net/http` client**

### Rationale
- vTeam already implements Jira Cloud and Server/Data Center support with dual authentication
- Direct HTTP requests provide full control and transparency
- Current implementation supports issue creation, updates, attachments, and remote links
- No changes needed to authentication infrastructure

### Existing Patterns to Reuse

**Authentication** (`/components/backend/jira/integration.go:306-414`):
- Credentials from Kubernetes Secret: `JIRA_URL`, `JIRA_PROJECT`, `JIRA_API_TOKEN`
- Jira Cloud: `Basic` auth with base64-encoded `email:api_token`
- Jira Server/DC: `Bearer` token
- Auto-detection based on "atlassian.net" in URL

**HTTP Client**:
- Standard operations: 30-second timeout
- File attachments: 60-second timeout

**Existing Operations**:
- Create issue: POST `/rest/api/2/issue`
- Update issue: PUT `/rest/api/2/issue/{key}`
- Attach file: POST `/rest/api/2/issue/{key}/attachments` (with `X-Atlassian-Token: no-check` header)

**Deduplication Pattern**:
- Current: Path-based lookup in RFEWorkflow CR's `jiraLinks` array
- BugFix adaptation: GitHub Issue number-based lookup

### Implementation Requirements for BugFix Workspace

**New Functions Needed** (add to `/components/backend/jira/integration.go`):

1. **CreateJiraTaskFromGitHubIssue**:
   - Input: GitHub Issue (title, description, URL), Jira auth header, Jira base URL, project key
   - Fields: `project`, `summary`, `description` (with GitHub Issue link at top), `issuetype: "Task"`
   - Output: Jira Task key (e.g., "PROJ-1234")

2. **UpdateJiraTask**:
   - Input: Jira key, updated description, auth header, Jira base URL
   - PUT `/rest/api/2/issue/{key}` with partial field update (description only)

3. **AddJiraComment**:
   - Input: Jira key, comment body (bugfix.md content), auth header, Jira base URL
   - POST `/rest/api/2/issue/{key}/comment`
   - Body: `{"body": "..."}`

4. **AddJiraRemoteLink** (bidirectional linking):
   - Input: Jira key, GitHub Issue URL, auth header, Jira base URL
   - POST `/rest/api/2/issue/{key}/remotelink`
   - Body: `{"object": {"url": "...", "title": "GitHub Issue #123", "icon": {"url16x16": "https://github.com/favicon.ico"}}}`

5. **Deduplication Logic**:
```go
type BugFixWorkflow struct {
    GithubIssueNumber int    `json:"githubIssueNumber"`
    JiraTaskKey       string `json:"jiraTaskKey,omitempty"`  // Empty until first sync
    LastSyncedAt      string `json:"lastSyncedAt,omitempty"`
}

// In sync handler:
if workflow.JiraTaskKey != "" {
    // Update existing Jira Task
} else {
    // Create new Jira Task, store key in CR
}
```

**Error Handling**:
- 401/403: Authentication failure (check `JIRA_API_TOKEN`)
- 404: Project not found (verify `JIRA_PROJECT` config)
- 429: Rate limit exceeded (read `Retry-After` header)
- Pass through Jira API error messages to frontend

**Bidirectional Linking Strategy**:
1. GitHub Issue → Jira Task: Use Jira Remote Links API (recommended) or include in description
2. Jira Task → GitHub Issue: Add GitHub Issue comment with Jira Task URL using GitHub API

### Alternatives Considered
- **Automatic bidirectional sync**: Rejected; manual "Sync to Jira" button provides better user control
- **go-jira library**: Rejected; direct HTTP provides more flexibility and consistency with codebase

---

## 3. Kubernetes CRD Design

### Decision
**Create `BugFixWorkflow` CRD following the established `RFEWorkflow` pattern**

### Rationale
- Existing RFE Workflow CRD provides proven template for workspace metadata
- Kubernetes Custom Resources are the primary source of truth for workspace state
- Simple two-phase status model ("Initializing", "Ready") is sufficient
- Namespace-scoped resources align with project-based organization

### CRD Structure

**Group**: `vteam.ambient-code`
**Version**: `v1alpha1`
**Kind**: `BugFixWorkflow`
**Scope**: Namespaced (one per project)

**Spec Fields**:
```yaml
spec:
  # Core bug information
  githubIssueNumber: 1234              # Required: GitHub Issue number
  githubIssueURL: "https://..."        # Required: Full GitHub Issue URL
  title: "Fix authentication bug"     # Required: Bug title
  description: "Bug description..."    # Required: Bug description

  # Git repository configuration
  umbrellaRepo:                        # Required: Spec repository
    url: "https://github.com/org/specs-repo"
    branch: "main"                     # Base branch (default: main)
  supportingRepos:                     # Optional: Supporting repos where fix is implemented
    - url: "https://github.com/org/backend-repo"
      branch: "main"

  # Feature branch configuration
  branchName: "bugfix/gh-1234"         # Platform-generated or user-provided

  # Jira integration (optional)
  jiraTaskKey: "PROJ-5678"             # Populated after first Jira sync
  lastSyncedAt: "2025-10-31T12:34:56Z" # Last Jira sync timestamp

  # Workspace filesystem path
  workspacePath: "/workspace/sessions/bugfix-1234/workspace"
```

**Status Fields**:
```yaml
status:
  phase: "Ready"                       # "Initializing" | "Ready"
  message: "Workspace ready"           # Human-readable status message
  bugFolderCreated: true               # Boolean: bug-{issue-number}/ exists in spec repo
  bugfixMarkdownCreated: true          # Boolean: bugfix-gh-{issue-number}.md exists
```

**Labels** (for querying):
- `project: <project-name>`
- `bugfix-issue-number: <issue-number>`

### Session Linking Pattern

**AgenticSession labels** (existing CRD, no changes needed):
```yaml
labels:
  project: <project-name>
  bugfix-workflow: <workflow-id>
  bugfix-session-type: "bug-review" | "bug-resolution-plan" | "bug-implement-fix" | "generic"
```

**Query pattern**:
```go
selector := fmt.Sprintf("bugfix-workflow=%s,project=%s", workflowID, projectName)
sessions, err := dynClient.Resource(agenticSessionGVR).Namespace(projectName).
    List(ctx, v1.ListOptions{LabelSelector: selector})
```

### Implementation Files

1. **CRD Definition**: `/components/manifests/crds/bugfixworkflows-crd.yaml`
2. **Type Definition**: `/components/backend/types/bugfix.go`
3. **CRD Operations**: `/components/backend/crd/bugfix.go`

**Functions to implement** (mirror `/components/backend/crd/rfe.go`):
- `GetProjectBugFixWorkflowCR(dynClient, project, id) (*types.BugFixWorkflow, error)`
- `ListProjectBugFixWorkflowCRs(dynClient, project) ([]types.BugFixWorkflow, error)`
- `UpsertProjectBugFixWorkflowCR(dynClient, workflow) error`
- `DeleteProjectBugFixWorkflowCR(dynClient, project, id) error`

### Alternatives Considered
- **Extend RFEWorkflow CRD**: Rejected; separate CRD provides clearer separation and avoids field pollution
- **Store in database**: Rejected; Kubernetes etcd is already the source of truth for all workspaces

---

## 4. WebSocket Event Patterns

### Decision
**Reuse existing WebSocket infrastructure with bug-specific event types**

### Rationale
- vTeam already has WebSocket server implementation using `gorilla/websocket`
- Real-time updates are essential for session status and progress tracking
- No changes needed to WebSocket infrastructure; only add new event types

### Existing WebSocket Patterns

**WebSocket Endpoint**: `/ws` (existing handler in `/components/backend/websocket/`)

**Connection Management**:
- Clients connect to `/ws` with project context
- Server maintains connection pool
- Automatic reconnection on disconnect

**Event Broadcasting Pattern**:
```go
// Broadcast event to all clients in project
wsServer.BroadcastToProject(projectName, Event{
    Type: "bugfix-session-started",
    Data: map[string]interface{}{
        "workflowId": workflowID,
        "sessionId": sessionID,
        "sessionType": "bug-review",
        "timestamp": time.Now().UTC().Format(time.RFC3339),
    },
})
```

### Implementation Requirements

**New Event Types to Add**:
1. `bugfix-workspace-created` - Workspace creation complete
2. `bugfix-session-started` - Bug-review/resolution-plan/implement-fix session started
3. `bugfix-session-progress` - Session progress updates (analysis complete, fix implemented, etc.)
4. `bugfix-session-completed` - Session finished successfully
5. `bugfix-session-failed` - Session encountered error
6. `bugfix-jira-sync-started` - Jira synchronization initiated
7. `bugfix-jira-sync-completed` - Jira sync successful
8. `bugfix-jira-sync-failed` - Jira sync failed

**Event Payload Structure**:
```go
type BugFixEvent struct {
    Type      string                 `json:"type"`
    Timestamp string                 `json:"timestamp"`
    Data      map[string]interface{} `json:"data"`
}

// Example: bugfix-session-progress
{
    "type": "bugfix-session-progress",
    "timestamp": "2025-10-31T12:34:56Z",
    "data": {
        "workflowId": "bugfix-1234",
        "sessionId": "session-abc",
        "sessionType": "bug-review",
        "phase": "analyzing",
        "message": "Analyzing codebase for root cause...",
        "progress": 0.4
    }
}
```

**Frontend Integration** (Next.js):
```typescript
// Hook: useBugFixWebSocket
const ws = useWebSocket(`/ws?project=${projectName}`);

ws.on('bugfix-session-progress', (event) => {
    updateSessionProgress(event.data);
});
```

### Alternatives Considered
- **Server-Sent Events (SSE)**: Rejected; WebSocket already implemented and provides bidirectional communication
- **Polling**: Rejected; WebSocket provides real-time updates with lower latency

---

## 5. Spec Repository Git Operations

### Decision
**Use existing `os/exec`-based git CLI operations (no third-party library)**

### Rationale
- vTeam already implements comprehensive Git operations using git CLI via `os/exec`
- Production-proven patterns for clone, commit, push workflows
- Token injection and authentication infrastructure exists
- No dependency on third-party libraries (simpler, more maintainable)
- Shallow clones and temporary directories provide excellent performance

### Existing Infrastructure

**Location**: `/components/backend/git/operations.go`

**Key Functions Available**:
1. **GetGitHubToken** (lines 77-147): Unified token acquisition (GitHub App → fallback to project secret)
2. **InjectGitHubToken** (lines 648-661): Insert token into git URL for authentication
3. **PerformRepoSeeding** (lines 296-645): Clone, branch, modify files, commit, push pattern
4. **PushRepo** (lines 694-828): Stage all changes, commit, push to remote
5. **CheckBranchExists** (lines 935-967): Verify branch exists via GitHub API
6. **ReadGitHubFile** (lines 908-932): Read file content from GitHub via API
7. **ValidatePushAccess** (lines 970-1034): Pre-validate write permissions before operations

**Authentication Patterns**:
```go
// Method 1: URL injection (for clone)
authenticatedURL, err := InjectGitHubToken(repoURL, token)
// Result: https://x-access-token:TOKEN@github.com/owner/repo.git

// Method 2: Git config (for push)
cfg := fmt.Sprintf("url.https://x-access-token:%s@github.com/.insteadOf=https://github.com/", token)
exec.CommandContext(ctx, "git", "-c", cfg, "push", "-u", "origin", branch)
```

**Performance Optimizations**:
- Shallow clones: `git clone --depth 1 --branch <branch> <url>`
- Temporary directories with cleanup: `os.MkdirTemp("", "bugfix-*")` + `defer os.RemoveAll()`
- Context-based cancellation: `exec.CommandContext(ctx, ...)`
- Token caching via TokenManager

### Implementation Requirements for BugFix Workspace

**Create new package**: `/components/backend/bugfix/git_operations.go`

**Functions to implement**:

1. **CreateBugFolder**:
   - Clone spec repo (shallow clone, single branch)
   - Create `bug-<issue-number>/` directory
   - Create initial `README.md` in folder
   - Commit and push changes
   - Reuse: `GetGitHubToken`, `InjectGitHubToken`, `PushRepo`, `ValidatePushAccess`

2. **CreateOrUpdateBugfixMarkdown**:
   - Clone spec repo
   - Check if `bug-<issue-number>/bugfix-gh-<issue-number>.md` exists
   - Create new file or append to existing content
   - Commit and push changes
   - Reuse: Same helpers as CreateBugFolder

3. **GetBugfixContent**:
   - Use `ReadGitHubFile` to fetch existing bugfix.md content via API
   - No clone needed (faster)
   - Reuse: `ReadGitHubFile`

4. **CheckBugFolderExists**:
   - Use GitHub API to check if `bug-<issue-number>/` path exists
   - No clone needed
   - Reuse: `checkGitHubPathExists` (lines 252-279)

**Error Handling Patterns**:
```go
// Pre-validation before operations
if err := ValidatePushAccess(ctx, specRepoURL, token); err != nil {
    return fmt.Errorf("cannot write to spec repo: %w. Check repository permissions", err)
}

// Detailed error messages with git output
cmd := exec.CommandContext(ctx, "git", cloneArgs...)
if out, err := cmd.CombinedOutput(); err != nil {
    outStr := string(out)
    if strings.Contains(outStr, "Authentication failed") {
        return fmt.Errorf("authentication failed: token may be expired or invalid")
    }
    if strings.Contains(outStr, "not found") {
        return fmt.Errorf("branch '%s' not found in repository", branchName)
    }
    return fmt.Errorf("git clone failed: %w (output: %s)", err, outStr)
}

// Cleanup on failure
defer func() {
    if removeErr := os.RemoveAll(repoDir); removeErr != nil {
        log.Printf("Warning: failed to cleanup %s: %v", repoDir, removeErr)
    }
}()
```

**Concurrency Strategy**: Session-level isolation
- Each session uses unique temporary directory
- No shared state between concurrent operations
- Git's optimistic concurrency (push conflicts handled by retry)

### Alternatives Considered
- **go-git library**: Rejected; adds 20MB to binary, slower than CLI, not used in codebase
- **git2go (libgit2 bindings)**: Rejected; CGO dependency complicates builds, platform-specific issues

---

## 6. Session Orchestration Patterns

### Decision
**Reuse existing AgenticSession CRD and operator infrastructure**

### Rationale
- AgenticSession CRD already handles session lifecycle (Pending → Creating → Running → Completed/Failed)
- Operator watches AgenticSessions and creates Kubernetes Jobs with two containers (content service + Claude runner)
- No changes needed to operator or session CRD
- Only need to add bug-specific labels for querying

### Existing Session Architecture

**AgenticSession CRD** (`/components/manifests/crds/agenticsessions-crd.yaml`):
- Independent from workspace CRDs
- Linked to workspaces via labels
- Operator-managed lifecycle

**Operator Pattern** (`/components/operator/internal/handlers/sessions.go`):
1. Watch for AgenticSession events (Added, Modified)
2. When phase = "Pending", create Kubernetes Job
3. Job contains two containers:
   - `ambient-content`: Content service (keeps pod alive, serves files via HTTP)
   - `ambient-code-runner`: Claude Code execution container
4. Monitor job status, update AgenticSession status
5. On completion/failure, cleanup job and per-job service

**Environment Variables Passed to Sessions**:
```go
environmentVariables := map[string]string{
    "REPOS_JSON": "<serialized-repos>",
    "AGENT_PERSONA": "stella",  // or AGENT_PERSONAS: "stella,alex"
    "PROJECT_NAME": projectName,
    "USER_ID": userID,
    // Add bug-specific vars:
    "GITHUB_ISSUE_NUMBER": "1234",
    "GITHUB_ISSUE_URL": "https://...",
    "BUGFIX_WORKFLOW_ID": "bugfix-1234",
    "SESSION_TYPE": "bug-review" | "bug-resolution-plan" | "bug-implement-fix" | "generic",
}
```

**Session Creation Pattern** (from `/components/backend/handlers/sessions.go`):
```go
// Create session linked to BugFix workspace
spec := map[string]interface{}{
    "title": "Bug Review Session for Issue #1234",
    "description": "Analyze bug and identify root cause",
    "repos": repos,
    "environmentVariables": envVars,
}

labels := map[string]interface{}{
    "project": projectName,
    "bugfix-workflow": bugfixWorkflowID,
    "bugfix-session-type": "bug-review",
}

sessionObj := &unstructured.Unstructured{
    Object: map[string]interface{}{
        "apiVersion": "vteam.ambient-code/v1alpha1",
        "kind": "AgenticSession",
        "metadata": map[string]interface{}{
            "name": sessionName,
            "namespace": projectName,
            "labels": labels,
        },
        "spec": spec,
    },
}

// Operator watches and creates Job automatically
```

**Feature Branch Propagation** (from RFE pattern):
```go
// All sessions linked to BugFix workspace use same feature branch
if bugfixWorkflowID != "" {
    bugfixWf, _ := GetProjectBugFixWorkflowCR(dynClient, project, bugfixWorkflowID)
    if bugfixWf != nil && bugfixWf.BranchName != "" {
        // Override all repo branches to use feature branch
        for i := range repos {
            repos[i]["input"].(map[string]interface{})["branch"] = bugfixWf.BranchName
            repos[i]["output"].(map[string]interface{})["branch"] = bugfixWf.BranchName
        }
    }
}
```

### Implementation Requirements

**No operator changes needed**. Only add handlers for session creation:

1. **CreateBugReviewSession**:
   - Handler: `POST /api/projects/:projectName/bugfix-workflows/:id/sessions/bug-review`
   - Create AgenticSession with labels linking to BugFix workspace
   - Pass environment variables: `GITHUB_ISSUE_NUMBER`, `SESSION_TYPE: "bug-review"`

2. **CreateBugResolutionPlanSession**:
   - Handler: `POST /api/projects/:projectName/bugfix-workflows/:id/sessions/bug-resolution-plan`
   - Same pattern as bug-review

3. **CreateBugImplementFixSession**:
   - Handler: `POST /api/projects/:projectName/bugfix-workflows/:id/sessions/bug-implement-fix`
   - Same pattern

4. **CreateGenericSession**:
   - Handler: `POST /api/projects/:projectName/bugfix-workflows/:id/sessions/generic`
   - Open-ended session without structured constraints

**List Sessions for BugFix Workspace**:
```go
// Handler: GET /api/projects/:projectName/bugfix-workflows/:id/sessions
selector := fmt.Sprintf("bugfix-workflow=%s,project=%s", workflowID, projectName)
sessions, err := dynClient.Resource(agenticSessionGVR).Namespace(projectName).
    List(ctx, v1.ListOptions{LabelSelector: selector})
```

### Alternatives Considered
- **Create separate BugFixSession CRD**: Rejected; AgenticSession is generic and sufficient
- **Inline session execution**: Rejected; Kubernetes Jobs provide isolation, resource limits, monitoring

---

## 7. Agent Integration Patterns

### Decision
**Reuse existing agent discovery and invocation infrastructure**

### Rationale
- Agents are stored in `.claude/agents/` directory of umbrella (spec) repository
- Agent files are markdown with YAML frontmatter
- Frontend fetches agents via backend API (backend reads from GitHub API)
- Agent personas passed to sessions via environment variables
- No changes needed to agent infrastructure

### Existing Agent Patterns

**Agent File Format** (`.claude/agents/<persona-name>.md`):
```markdown
---
name: Stella
role: Staff Engineer
description: Staff Engineer focused on technical leadership and implementation
---

# Stella - Staff Engineer Agent

Technical instructions and context for Stella...
```

**Agent Discovery** (`/components/backend/handlers/rfe.go:GetProjectRFEWorkflowAgents`):
1. Parse umbrella repo URL to get owner/repo
2. Use feature branch (not base branch)
3. Fetch `.claude/agents/` directory via GitHub Contents API
4. Parse YAML frontmatter from each agent file
5. Return array of agents with persona, name, role, description

**Agent Invocation** (via environment variables):
```go
// Single agent
environmentVariables["AGENT_PERSONA"] = "stella"

// Multiple agents
environmentVariables["AGENT_PERSONAS"] = "stella,alex"
```

**Frontend Agent Selection** (`/components/frontend/src/app/projects/[name]/rfe/[id]/page.tsx`):
```typescript
const [selectedAgents, setSelectedAgents] = useState<string[]>([]);

<RfeAgentsCard
  projectName={project}
  workflowId={id}
  selectedAgents={selectedAgents}
  onAgentsChange={setSelectedAgents}
/>
```

### Implementation Requirements

**Backend Handler** (mirror RFE pattern):
```go
// GET /api/projects/:projectName/bugfix-workflows/:id/agents
func GetProjectBugFixWorkflowAgents(c *gin.Context) {
    // Get BugFix workflow
    bugfixWf := GetProjectBugFixWorkflowCR(...)

    // Parse umbrella repo URL
    owner, repo, _ := git.ParseGitHubURL(bugfixWf.UmbrellaRepo.URL)

    // Use feature branch
    ref := bugfixWf.BranchName

    // Get GitHub token
    token, _ := git.GetGitHubToken(ctx, k8sClient, dynClient, project, userID)

    // Fetch agents from .claude/agents/
    agents, _ := fetchAgentsFromRepo(ctx, owner, repo, ref, token)

    c.JSON(http.StatusOK, agents)
}
```

**Frontend Component** (reuse `RfeAgentsCard` or create `BugFixAgentsCard`):
```typescript
<BugFixAgentsCard
  projectName={projectName}
  workflowId={bugfixWorkflowId}
  selectedAgents={selectedAgents}
  onAgentsChange={setSelectedAgents}
/>
```

**Bug-Specific Agents to Seed** (in `.claude/agents/` directory):
1. `debugger.md` - Specializes in root cause analysis and debugging
2. `test-engineer.md` - Focuses on test creation and validation
3. `reviewer.md` - Reviews bug fixes for correctness and edge cases

### Alternatives Considered
- **Hardcode bug-specific agents**: Rejected; storing in repo provides flexibility and version control
- **Separate agent registry service**: Rejected; current Git-based approach is simple and sufficient

---

## 8. Technology Stack Summary

### Backend Stack
- **Language**: Go 1.24.0
- **Web Framework**: Gin (github.com/gin-gonic/gin v1.10.1)
- **Kubernetes Client**: client-go v0.34.0
- **JWT**: github.com/golang-jwt/jwt/v5 v5.3.0
- **WebSocket**: github.com/gorilla/websocket v1.5.4
- **Git Operations**: `os/exec` with git CLI (no library)

### Frontend Stack
- **Framework**: Next.js 15+
- **Runtime**: React 19
- **UI Components**: Radix UI
- **Styling**: Tailwind CSS
- **Data Fetching**: React Query (TanStack Query)
- **WebSocket**: Native WebSocket API or library wrapper

### Infrastructure
- **Kubernetes**: Custom Resource Definitions (CRDs) for workspace state
- **Orchestration**: Custom Kubernetes operator for session management
- **Storage**: Git repositories (no Persistent Volumes for workspaces)
- **Authentication**: GitHub App installation tokens + fallback to project secrets

---

## 9. Performance and Scale Targets

### From Feature Spec (Success Criteria)

| **Metric** | **Target** | **Implementation Notes** |
|------------|-----------|-------------------------|
| Workspace creation | < 5s | Reuse RFE creation pattern (CR creation + seeding) |
| GitHub API calls | < 2s | Use existing token caching, parallel requests where possible |
| Jira sync | < 10s | Single API call, no retry logic needed for happy path |
| WebSocket updates | < 100ms | Existing WebSocket infrastructure is fast |
| Concurrent workspaces | 100+ | CRs scale well, Git operations are isolated per session |
| Simultaneous WebSocket connections | 50+ | Existing implementation handles this |

### GitHub API Rate Limits
- **Authenticated requests**: 5,000/hour per user
- **Token caching**: Reduces token mint API calls
- **Pre-validation**: Avoids wasted API calls on invalid operations

### Jira API Rate Limits
- **Varies by Jira plan**: Typically 100-300 requests/minute
- **Manual sync only**: No automatic polling, reduces API usage
- **Error handling**: Return `Retry-After` header value on 429 errors

### Git Operations Performance
- **Shallow clones**: Only fetch single branch, single commit history
- **Temporary directories**: Fast cleanup, no persistent state
- **Context timeouts**: 15-30 seconds for git operations
- **Concurrent access**: Session-level isolation prevents conflicts

---

## 10. Security Considerations

### Token Management
- **GitHub App private key**: Stored in environment variable, supports PKCS1/PKCS8 formats
- **Installation tokens**: Short-lived (1 hour), cached until 3 minutes before expiry
- **Project secrets**: Stored in Kubernetes Secrets, namespace-scoped
- **Token injection**: URL-based or git config, never logged to stdout/stderr

### RBAC and Authorization
- **Request-scoped K8s clients**: All operations use authenticated user's RBAC permissions
- **Namespace isolation**: Workspaces and sessions are namespace-scoped per project
- **GitHub permissions**: Validate push access before git operations

### Data Protection
- **No sensitive data in CRs**: GitHub tokens never stored in Custom Resources
- **URL sanitization**: Git URLs sanitized in logs to remove tokens
- **Temporary directories**: Cleaned up after operations, no data leak

---

## 11. Open Questions (Resolved)

All technical unknowns from the plan.md Technical Context section have been resolved:

| **Unknown** | **Resolution** |
|-------------|----------------|
| GitHub API patterns | Use existing `github/token.go` infrastructure, create new `github/issues.go` |
| Jira API patterns | Use existing `jira/integration.go`, add comment and remote link functions |
| Kubernetes CRD design | Mirror `RFEWorkflow` CRD with bug-specific fields |
| WebSocket events | Reuse existing WebSocket infrastructure, add bug-specific event types |
| Spec Repository Git operations | Use existing `git/operations.go` with `os/exec`, create `bugfix/git_operations.go` |
| Session orchestration | Reuse existing `AgenticSession` CRD and operator, link via labels |
| Authentication/authorization | Use existing `GetGitHubToken` and request-scoped K8s clients |

---

## 12. Next Steps (Phase 1: Design & Contracts)

With all research complete, proceed to Phase 1:

1. **Generate data-model.md**: Define BugFix Workspace, Session, GitHub Issue, Jira Task entities
2. **Generate API contracts**: OpenAPI spec for BugFix workspace CRUD and session operations
3. **Generate quickstart.md**: Integration test scenarios from feature spec acceptance scenarios
4. **Update CLAUDE.md**: Add BugFix Workspace context for future development sessions

---

**Research Status**: ✅ COMPLETE
**Phase 0 Output**: This document (research.md)
**Ready for Phase 1**: YES
