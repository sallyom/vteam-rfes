# Data Model: BugFix Workspace

**Date**: 2025-10-31
**Feature**: BugFix Workspace Type

---

## Overview

This document defines the data entities, relationships, validation rules, and state transitions for the BugFix Workspace feature. All entities follow patterns established by the existing RFE Workspace implementation in vTeam.

---

## Entity Definitions

### 1. BugFixWorkflow

**Description**: Represents a bug fix workspace, the top-level container for all bug-related work.

**Storage**: Kubernetes Custom Resource (CRD)
- Group: `vteam.ambient-code`
- Version: `v1alpha1`
- Kind: `BugFixWorkflow`
- Scope: Namespaced (per project)

**Lifecycle**: User-initiated creation → Seeding → Session execution → Manual deletion

#### Fields

| **Field** | **Type** | **Required** | **Description** |
|-----------|----------|--------------|-----------------|
| `id` | string | Yes | Platform-generated identifier (format: `bugfix-{timestamp}`) |
| `githubIssueNumber` | integer | Yes | GitHub Issue number (e.g., 1234) |
| `githubIssueURL` | string | Yes | Full GitHub Issue URL |
| `title` | string | Yes | Bug title (min 5 chars) |
| `description` | string | Yes | Bug description (min 20 chars) |
| `umbrellaRepo` | Repository | Yes | Spec repository for documentation |
| `supportingRepos` | []Repository | No | Repositories where fix is implemented |
| `branchName` | string | Yes | Feature branch name (format: `bugfix/gh-{issue-number}`) |
| `jiraTaskKey` | string | No | Jira Task key (e.g., "PROJ-5678"), populated after first sync |
| `lastSyncedAt` | timestamp | No | Last Jira sync timestamp (RFC3339 format) |
| `workspacePath` | string | No | Filesystem path (used by sessions) |
| `createdAt` | timestamp | Yes | Creation timestamp (RFC3339 format) |
| `createdBy` | string | Yes | User ID who created the workspace |

#### Status Fields

| **Field** | **Type** | **Description** |
|-----------|----------|-----------------|
| `phase` | enum | "Initializing" \| "Ready" |
| `message` | string | Human-readable status message |
| `bugFolderCreated` | boolean | True if `bug-{issue-number}/` exists in spec repo |
| `bugfixMarkdownCreated` | boolean | True if `bugfix-gh-{issue-number}.md` exists |

#### Validation Rules

1. **githubIssueURL**: Must match pattern `https://github.com/{owner}/{repo}/issues/{number}`
2. **branchName**: Must not be empty, recommended format `bugfix/gh-{issue-number}`
3. **title**: Minimum 5 characters, maximum 200 characters
4. **description**: Minimum 20 characters, maximum 5000 characters
5. **umbrellaRepo.url**: Must be valid GitHub repository URL
6. **jiraTaskKey**: If present, must match pattern `{PROJECT}-{NUMBER}` (e.g., "PROJ-123")

#### Relationships

- **Has Many** AgenticSessions (linked via labels `bugfix-workflow={id}`)
- **Belongs To** Project (namespace-scoped)
- **References** GitHub Issue (external entity)
- **References** Jira Task (external entity, optional)

---

### 2. Repository

**Description**: Git repository reference for spec documentation or implementation code.

**Storage**: Embedded within BugFixWorkflow spec

#### Fields

| **Field** | **Type** | **Required** | **Description** |
|-----------|----------|--------------|-----------------|
| `url` | string | Yes | GitHub repository URL |
| `branch` | string | No | Base branch (default: "main") |

#### Validation Rules

1. **url**: Must be valid GitHub HTTPS URL (format: `https://github.com/{owner}/{repo}`)
2. **branch**: If provided, must not be empty, must not be protected branch (main, master, develop)

---

### 3. AgenticSession

**Description**: Execution unit for bug-related work (analysis, planning, implementation).

**Storage**: Kubernetes Custom Resource (existing CRD, no changes needed)
- Group: `vteam.ambient-code`
- Version: `v1alpha1`
- Kind: `AgenticSession`

**Note**: This is an existing entity. BugFix Workspace uses it via labels.

#### BugFix-Specific Labels

| **Label Key** | **Value** | **Description** |
|---------------|-----------|-----------------|
| `project` | string | Project name (namespace) |
| `bugfix-workflow` | string | BugFixWorkflow ID |
| `bugfix-session-type` | enum | "bug-review" \| "bug-resolution-plan" \| "bug-implement-fix" \| "generic" |

#### BugFix-Specific Environment Variables

| **Variable** | **Type** | **Description** |
|--------------|----------|-----------------|
| `GITHUB_ISSUE_NUMBER` | string | GitHub Issue number (e.g., "1234") |
| `GITHUB_ISSUE_URL` | string | Full GitHub Issue URL |
| `BUGFIX_WORKFLOW_ID` | string | BugFixWorkflow ID |
| `SESSION_TYPE` | string | "bug-review" \| "bug-resolution-plan" \| "bug-implement-fix" \| "generic" |

#### State Transitions

```
Pending → Creating → Running → (Completed | Failed | Stopped)
```

1. **Pending**: Session created, waiting for operator
2. **Creating**: Operator creating Kubernetes Job
3. **Running**: Job is executing
4. **Completed**: Job succeeded (exit code 0)
5. **Failed**: Job failed (exit code non-zero or backoff limit reached)
6. **Stopped**: User manually stopped session

---

### 4. GitHub Issue

**Description**: External entity representing a GitHub Issue. Source of truth for bug information.

**Storage**: GitHub.com (external)

**Access Pattern**: Via GitHub REST API v3

#### Fields (Relevant Subset)

| **Field** | **Type** | **Description** |
|-----------|----------|-----------------|
| `number` | integer | Issue number (unique within repository) |
| `url` | string | Issue URL (e.g., `https://github.com/owner/repo/issues/123`) |
| `title` | string | Issue title |
| `body` | string | Issue description (markdown) |
| `state` | enum | "open" \| "closed" |
| `labels` | []string | Issue labels |
| `assignees` | []string | Assigned users |

#### Operations from BugFix Workspace

1. **Create**: Workspace creates issue from text description
2. **Read**: Workspace loads issue details by URL or number
3. **Update**: Workspace adds comments with analysis, plan, implementation summary
4. **Link**: Workspace adds Jira Task link as comment (if Jira sync used)

#### Validation Rules

1. Issue must exist and be accessible by authenticated user
2. Repository must allow issue creation (for text description flow)

---

### 5. Jira Task

**Description**: External entity representing a Jira Task/Issue. Optional secondary tracking artifact.

**Storage**: Jira Cloud or Server/Data Center (external)

**Access Pattern**: Via Jira REST API v2

#### Fields (Relevant Subset)

| **Field** | **Type** | **Description** |
|-----------|----------|-----------------|
| `key` | string | Task key (e.g., "PROJ-1234") |
| `url` | string | Task URL (e.g., `https://company.atlassian.net/browse/PROJ-1234`) |
| `summary` | string | Task title |
| `description` | string | Task description (Jira markup) |
| `status` | string | Task status (project-specific statuses) |
| `issueType` | string | "Task" (for bug fixes) |

#### Operations from BugFix Workspace

1. **Create**: "Sync to Jira" button creates Jira Task from GitHub Issue
2. **Update**: Subsequent syncs update description (not summary)
3. **Add Comment**: Bug-implement-fix session posts bugfix.md content as comment
4. **Add Remote Link**: Link to GitHub Issue URL

#### Deduplication Logic

- BugFixWorkflow stores `jiraTaskKey` after first sync
- Subsequent syncs update existing task instead of creating new one

---

### 6. Bug Documentation Folder

**Description**: Folder structure in spec repository containing all bug-related documentation.

**Storage**: Git repository (umbrellaRepo in BugFixWorkflow)

**Location**: `bug-{issue-number}/` (e.g., `bug-1234/`)

#### Contents

| **File** | **Required** | **Description** |
|----------|--------------|-----------------|
| `bugfix-gh-{issue-number}.md` | Yes | Primary artifact: implementation plan, steps, PR links |
| `README.md` | Yes | Folder overview (auto-created) |
| Additional files | No | Screenshots, logs, test results (user-added) |

#### Creation Pattern

1. Created during workspace initialization (after GitHub Issue validated)
2. Created via git operations: clone → create folder → commit → push
3. Feature branch used (not base branch)

---

### 7. Bug Documentation Markdown

**Description**: Primary documentation artifact for bug fix.

**Storage**: Git repository at `bug-{issue-number}/bugfix-gh-{issue-number}.md`

#### Structure

```markdown
# Bug Fix: {GitHub Issue Title}

**GitHub Issue**: {GitHub Issue URL}
**Jira Task**: {Jira Task URL} *(if synced)*
**Status**: Open | In Progress | Fixed | Verified

---

## Root Cause Analysis
*(Updated by Bug-review session)*

{Technical analysis findings, affected components, root cause}

---

## Resolution Plan
*(Updated by Bug-resolution-plan session)*

{Proposed fix approach, alternatives considered, implementation strategy}

---

## Implementation Steps
*(Updated by Bug-implement-fix session)*

1. Step 1: {description}
2. Step 2: {description}
...

**Feature Branch**: `bugfix/gh-{issue-number}`
**Pull Request**: {PR URL}

---

## Testing
*(Updated by Bug-implement-fix session)*

- Test 1: {description}
- Test 2: {description}

---

## Additional Notes
*(User or agent notes)*

{Any additional context, edge cases, future improvements}
```

#### Update Pattern

- **Bug-review session**: Appends "Root Cause Analysis" section
- **Bug-resolution-plan session**: Appends "Resolution Plan" section
- **Bug-implement-fix session**: Appends "Implementation Steps", "Testing", PR link

---

## Entity Relationships

```
Project (Namespace)
  └── BugFixWorkflow (CRD)
        ├── References GitHub Issue (external)
        ├── References Jira Task (external, optional)
        ├── Has umbrellaRepo (Repository)
        ├── Has supportingRepos[] (Repository)
        ├── Creates bug-{issue-number}/ folder (Git)
        │     └── Contains bugfix-gh-{issue-number}.md (Git)
        └── Links to AgenticSessions (CRD, via labels)
              ├── bug-review session
              ├── bug-resolution-plan session
              ├── bug-implement-fix session
              └── generic session(s)
```

**Cardinality**:
- **BugFixWorkflow : GitHub Issue** = 1:1 (one workflow per issue)
- **BugFixWorkflow : Jira Task** = 1:0..1 (optional Jira sync)
- **BugFixWorkflow : AgenticSession** = 1:N (multiple sessions per workflow)
- **BugFixWorkflow : Bug Folder** = 1:1 (one folder per workflow)
- **Bug Folder : Bug Documentation Markdown** = 1:1 (primary artifact)

---

## State Transitions

### BugFixWorkflow State Machine

```
[User Creates] → Initializing
                     ↓
               [Validation Complete]
                     ↓
                   Ready
                     ↓
             [User Deletes] → (Deleted)
```

**State Descriptions**:
1. **Initializing**: Validating GitHub Issue, creating branch (if needed), creating bug folder
2. **Ready**: Workspace ready for session execution
3. **(Deleted)**: User manually deleted workspace (no auto-cleanup)

**Allowed Transitions**:
- Initializing → Ready (success)
- Initializing → (Deleted) (user cancels during initialization)
- Ready → (Deleted) (user deletes workspace)

**Disallowed Transitions**:
- Ready → Initializing (cannot re-initialize)

---

### AgenticSession State Machine (Existing)

```
[Workspace Creates Session] → Pending
                                 ↓
                      [Operator Creates Job]
                                 ↓
                             Creating
                                 ↓
                      [Job Pod Running]
                                 ↓
                             Running
                                 ↓
              ┌──────────────────┼──────────────────┐
              ↓                  ↓                  ↓
         Completed           Failed             Stopped
```

**State Descriptions**:
1. **Pending**: AgenticSession CR created, waiting for operator
2. **Creating**: Operator creating Kubernetes Job and PVC
3. **Running**: Job pod is executing Claude Code
4. **Completed**: Job succeeded (exit code 0)
5. **Failed**: Job failed (exit code non-zero, backoff limit reached)
6. **Stopped**: User manually stopped session

---

## Data Validation Summary

### Workspace Creation Validation

| **Field** | **Validation** | **Error Message** |
|-----------|----------------|-------------------|
| `githubIssueURL` | Must be accessible via GitHub API | "GitHub Issue not found or inaccessible" |
| `title` | 5-200 characters | "Title must be between 5 and 200 characters" |
| `description` | 20-5000 characters | "Description must be between 20 and 5000 characters" |
| `umbrellaRepo.url` | Valid GitHub URL, push access | "Invalid repository URL or no push access" |
| `branchName` | Not empty, not protected | "Branch name cannot be empty or protected" |

### Session Creation Validation

| **Field** | **Validation** | **Error Message** |
|-----------|----------------|-------------------|
| `workflowId` | BugFixWorkflow must exist | "BugFix Workspace not found" |
| `sessionType` | Must be valid enum value | "Invalid session type" |
| `repos` | Must include umbrella repo | "Umbrella repository required" |

### Jira Sync Validation

| **Field** | **Validation** | **Error Message** |
|-----------|----------------|-------------------|
| `JIRA_URL` | Must be set in project secret | "Jira URL not configured" |
| `JIRA_PROJECT` | Must be set in project secret | "Jira project not configured" |
| `JIRA_API_TOKEN` | Must be set in project secret | "Jira API token not configured" |

---

## Data Access Patterns

### Read Operations

1. **Get BugFixWorkflow by ID**:
   - Query: `kubectl get bugfixworkflow {id} -n {project}`
   - API: `GET /api/projects/{project}/bugfix-workflows/{id}`
   - Use case: Workspace detail page, session creation

2. **List BugFixWorkflows in Project**:
   - Query: `kubectl get bugfixworkflows -n {project}`
   - API: `GET /api/projects/{project}/bugfix-workflows`
   - Use case: Workspace list page

3. **List Sessions for BugFixWorkflow**:
   - Query: `kubectl get agenticsessions -n {project} -l bugfix-workflow={id}`
   - API: `GET /api/projects/{project}/bugfix-workflows/{id}/sessions`
   - Use case: Session list in workspace detail page

4. **Get Bug Documentation Content**:
   - GitHub API: `GET /repos/{owner}/{repo}/contents/bug-{issue-number}/bugfix-gh-{issue-number}.md`
   - Use case: Display bugfix.md in UI

### Write Operations

1. **Create BugFixWorkflow**:
   - API: `POST /api/projects/{project}/bugfix-workflows`
   - Side effects:
     - Validate GitHub Issue
     - Create bug folder in spec repo
     - Create BugFixWorkflow CR
     - Set phase to "Ready"

2. **Update BugFixWorkflow**:
   - API: `PUT /api/projects/{project}/bugfix-workflows/{id}`
   - Use case: Update description, add Jira Task key after sync

3. **Delete BugFixWorkflow**:
   - API: `DELETE /api/projects/{project}/bugfix-workflows/{id}`
   - Side effects:
     - Delete CR (Kubernetes cascades to owned resources)
     - Note: Git branch NOT deleted (manual cleanup)

4. **Create Session**:
   - API: `POST /api/projects/{project}/bugfix-workflows/{id}/sessions`
   - Side effects:
     - Create AgenticSession CR with labels
     - Operator creates Kubernetes Job

5. **Sync to Jira**:
   - API: `POST /api/projects/{project}/bugfix-workflows/{id}/sync-jira`
   - Side effects:
     - Create or update Jira Task
     - Add remote link from Jira to GitHub Issue
     - Add GitHub Issue comment with Jira link
     - Update BugFixWorkflow CR with Jira Task key

---

## Indexing and Query Optimization

**Kubernetes Label Selectors**:
- `project={project-name}` - All workspaces in a project
- `bugfix-workflow={workflow-id}` - All sessions for a workflow
- `bugfix-session-type={session-type}` - Filter sessions by type
- `bugfix-issue-number={issue-number}` - Find workspace by issue number

**Frontend Query Patterns** (React Query):
- `useQuery(['bugfix-workflows', projectName])` - List workspaces
- `useQuery(['bugfix-workflow', projectName, workflowId])` - Get workspace
- `useQuery(['bugfix-sessions', projectName, workflowId])` - List sessions
- `useQuery(['bugfix-agents', projectName, workflowId])` - List agents

---

## Data Consistency Considerations

### Eventual Consistency

1. **Kubernetes CR Updates**: Status updates via Kubernetes API are eventually consistent
   - Mitigation: Use WebSocket events for real-time UI updates

2. **GitHub Issue Comments**: Added by multiple sessions, order not guaranteed
   - Mitigation: Timestamp each comment, client sorts by timestamp

3. **Git Operations**: Concurrent pushes to same branch may conflict
   - Mitigation: Session-level isolation, retry on push conflict

### Strong Consistency

1. **BugFixWorkflow CR**: Read-after-write consistency within Kubernetes
2. **Jira Task Creation**: Synchronous API call, immediate consistency
3. **GitHub Issue Creation**: Synchronous API call, immediate consistency

---

## Summary

The BugFix Workspace data model reuses proven patterns from RFE Workspace:
- Kubernetes CRDs for workspace state
- Existing AgenticSession infrastructure for execution
- Git repositories for documentation artifacts
- External integrations with GitHub and Jira

All entities have clear validation rules, state transitions, and relationships. The model supports the full workflow from bug identification through implementation and tracking.
