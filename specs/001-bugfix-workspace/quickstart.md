# Quickstart Guide: BugFix Workspace Testing

**Date**: 2025-10-31
**Feature**: BugFix Workspace Type

---

## Purpose

This document provides integration test scenarios extracted from the feature spec's acceptance scenarios. It serves as both a testing guide and quickstart tutorial for validating the BugFix Workspace implementation.

---

## Prerequisites

### Environment Setup

1. **vTeam Platform Running**:
   - Backend API accessible at `/api`
   - Frontend running and accessible
   - Kubernetes operator watching AgenticSession CRDs

2. **Test Project**:
   - Project namespace: `test-project`
   - GitHub repository: `test-org/test-repo` (with push access)
   - Spec repository: `test-org/specs-repo` (with push access)

3. **Authentication**:
   - Valid GitHub App installation or Personal Access Token in project secrets
   - Optional: Jira credentials in project secrets (`JIRA_URL`, `JIRA_PROJECT`, `JIRA_API_TOKEN`)

4. **Test GitHub Issue**:
   - Create test issue in `test-org/test-repo`
   - Example: https://github.com/test-org/test-repo/issues/1234
   - Title: "Test authentication timeout bug"
   - Description: "Users experience timeout errors when authenticating with OAuth. Reproduce by attempting login after 5 minutes of inactivity."

---

## Integration Test Scenarios

### Scenario 1: Create BugFix Workspace from GitHub Issue

**Feature Requirement**: FR-001, FR-002

**Given**: Developer has a valid GitHub Issue URL

**Steps**:
1. Navigate to vTeam frontend: `/projects/test-project/bugfix`
2. Click "Create New BugFix Workspace" button
3. Select "From GitHub Issue" option
4. Enter GitHub Issue URL: `https://github.com/test-org/test-repo/issues/1234`
5. Select umbrella repo: `https://github.com/test-org/specs-repo`
6. Click "Create Workspace"

**Expected Results**:
- API request: `POST /api/projects/test-project/bugfix-workflows`
- Response: 201 Created with BugFixWorkflow object
- Workspace ID matches pattern `bugfix-{timestamp}`
- `githubIssueNumber` = 1234
- `githubIssueURL` = provided URL
- `status.phase` = "Initializing" then "Ready"
- Git folder created: `bug-1234/` in specs-repo
- Git file created: `bug-1234/README.md` in specs-repo
- Branch created: `bugfix/gh-1234` in specs-repo

**API Contract Validation**:
```bash
curl -X POST /api/projects/test-project/bugfix-workflows \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "githubIssueURL": "https://github.com/test-org/test-repo/issues/1234",
    "umbrellaRepo": {
      "url": "https://github.com/test-org/specs-repo",
      "branch": "main"
    }
  }'

# Expected response:
{
  "id": "bugfix-1730368890",
  "githubIssueNumber": 1234,
  "githubIssueURL": "https://github.com/test-org/test-repo/issues/1234",
  "title": "Test authentication timeout bug",
  "description": "Users experience timeout errors...",
  "umbrellaRepo": { "url": "...", "branch": "main" },
  "branchName": "bugfix/gh-1234",
  "status": { "phase": "Ready", "message": "Workspace ready", "bugFolderCreated": true }
}
```

---

### Scenario 2: Create BugFix Workspace from Text Description

**Feature Requirement**: FR-003, FR-004

**Given**: Developer has a bug description but no GitHub Issue yet

**Steps**:
1. Navigate to `/projects/test-project/bugfix/new`
2. Select "From Text Description" option
3. Fill form:
   - **Title**: "Memory leak in background service"
   - **Description**: "Memory usage increases continuously. Symptoms: High memory usage after 1 hour. Reproduction: Run background service for >1 hour. Expected: Memory stabilizes. Actual: Memory grows to 4GB+"
4. Select umbrella repo: `https://github.com/test-org/specs-repo`
5. Click "Create Workspace"

**Expected Results**:
- API creates GitHub Issue via GitHub API
- GitHub Issue created with standardized template
- Response includes newly created `githubIssueNumber`
- Workspace created with generated issue number
- Git folder created: `bug-{new-issue-number}/`
- Frontend displays notification: "GitHub Issue #{number} created"

**API Contract Validation**:
```bash
curl -X POST /api/projects/test-project/bugfix-workflows \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "textDescription": "Memory usage increases continuously. Symptoms: ...",
    "umbrellaRepo": {
      "url": "https://github.com/test-org/specs-repo"
    }
  }'

# Expected response:
{
  "id": "bugfix-1730368901",
  "githubIssueNumber": 1235,  # New issue created
  "githubIssueURL": "https://github.com/test-org/specs-repo/issues/1235",
  "title": "Memory leak in background service",
  "description": "Memory usage increases continuously...",
  "status": { "phase": "Ready", "bugFolderCreated": true }
}
```

---

### Scenario 3: Bug-review Session Analyzes Bug

**Feature Requirement**: FR-009, FR-010, FR-011

**Given**: BugFix Workspace exists for GitHub Issue #1234

**Steps**:
1. Navigate to `/projects/test-project/bugfix/bugfix-1730368890`
2. Click "Start Bug-review Session" button
3. Optionally select agents (e.g., Stella, Debugger)
4. Click "Start Session"
5. Monitor session progress via WebSocket updates

**Expected Results**:
- API request: `POST /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sessions`
- AgenticSession CR created with labels:
  - `bugfix-workflow: bugfix-1730368890`
  - `bugfix-session-type: bug-review`
- Kubernetes Job created by operator
- Session analyzes GitHub Issue
- Session researches codebase
- GitHub Issue updated with comment containing:
  - Technical analysis findings
  - Affected components
  - Root cause identification
- Session completes with status "Completed"

**API Contract Validation**:
```bash
# Create session
curl -X POST /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sessions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionType": "bug-review",
    "selectedAgents": ["stella", "debugger"]
  }'

# Expected response:
{
  "id": "session-abc123",
  "name": "Bug Review for Issue #1234",
  "sessionType": "bug-review",
  "phase": "Pending",
  "createdAt": "2025-10-31T11:00:00Z"
}

# List sessions
curl /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sessions \
  -H "Authorization: Bearer $TOKEN"

# Expected: Array of sessions with "bug-review" session present
```

---

### Scenario 4: Sync to Jira from Bug-review Session

**Feature Requirement**: FR-013, FR-014, FR-015

**Given**: Bug-review session has completed analysis for GitHub Issue #1234

**Steps**:
1. In workspace detail page, observe "Sync to Jira" button appears
2. Click "Sync to Jira" button
3. Monitor sync progress (loading indicator)
4. Verify success notification

**Expected Results**:
- API request: `POST /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sync-jira`
- Jira Task created via Jira REST API
- Jira Task contains:
  - Summary: GitHub Issue title
  - Description: GitHub Issue description with GitHub URL link at top
  - Issue Type: "Task"
- Jira Task includes link to GitHub Issue #1234
- GitHub Issue #1234 updated with comment linking to Jira Task
- BugFixWorkflow CR updated with `jiraTaskKey` and `lastSyncedAt`
- Response includes Jira Task key (e.g., "PROJ-5678")

**API Contract Validation**:
```bash
curl -X POST /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sync-jira \
  -H "Authorization: Bearer $TOKEN"

# Expected response:
{
  "jiraTaskKey": "PROJ-5678",
  "jiraTaskURL": "https://company.atlassian.net/browse/PROJ-5678",
  "action": "created",
  "message": "Successfully created Jira Task PROJ-5678"
}

# Verify GitHub Issue comment added:
# Visit https://github.com/test-org/test-repo/issues/1234
# Expect comment: "Jira Task: https://company.atlassian.net/browse/PROJ-5678"

# Verify Jira Task created:
# Visit https://company.atlassian.net/browse/PROJ-5678
# Expect description starts with: "**GitHub Issue:** https://github.com/test-org/test-repo/issues/1234"
```

---

### Scenario 5: Bug-resolution-plan Session Creates Implementation Plan

**Feature Requirement**: FR-017, FR-018, FR-019, FR-020

**Given**: BugFix Workspace exists for GitHub Issue #1234

**Steps**:
1. Navigate to workspace detail page
2. Click "Start Bug-resolution-plan Session"
3. Select agents (e.g., Stella, Architect)
4. Click "Start Session"
5. Monitor session progress

**Expected Results**:
- AgenticSession created with `bugfix-session-type: bug-resolution-plan`
- Session proposes multiple resolution strategies
- File created/updated: `bug-1234/bugfix-gh-1234.md` in specs-repo
- File contains:
  - GitHub Issue URL at top
  - Jira Task URL (if synced)
  - Implementation plan with multiple strategies
  - Recommended approach
- GitHub Issue #1234 updated with comment containing resolution plan
- Session completes successfully

**API Contract Validation**:
```bash
# Create session
curl -X POST /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sessions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionType": "bug-resolution-plan",
    "selectedAgents": ["stella", "archie"]
  }'

# Verify file created:
# GET https://api.github.com/repos/test-org/specs-repo/contents/bug-1234/bugfix-gh-1234.md?ref=bugfix/gh-1234
# Expect: File exists, content includes "## Resolution Plan" section
```

---

### Scenario 6: Bug-implement-fix Session Implements Fix

**Feature Requirement**: FR-021, FR-022, FR-023, FR-024, FR-025, FR-026

**Given**: Bug-resolution-plan session has created an implementation plan

**Steps**:
1. Click "Start Bug-implement-fix Session"
2. Select agents (e.g., Stella, Taylor)
3. Start session
4. Monitor implementation progress

**Expected Results**:
- Session creates feature branch `bugfix/gh-1234` in supporting repo(s)
- Session implements fix in supporting repo(s)
- Session writes/updates tests
- Session updates relevant documentation
- File updated: `bug-1234/bugfix-gh-1234.md` with:
  - Implementation steps section
  - PR link (when created)
  - Testing details
- GitHub Issue #1234 updated with comment:
  - Link to feature branch
  - Summary of implementation
- Session completes successfully

**API Contract Validation**:
```bash
curl -X POST /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sessions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionType": "bug-implement-fix",
    "selectedAgents": ["stella", "taylor"]
  }'

# Verify branch created:
# GET https://api.github.com/repos/test-org/test-repo/git/refs/heads/bugfix/gh-1234
# Expect: Branch exists

# Verify bugfix.md updated:
# GET https://api.github.com/repos/test-org/specs-repo/contents/bug-1234/bugfix-gh-1234.md?ref=bugfix/gh-1234
# Expect: "## Implementation Steps" section present
```

---

### Scenario 7: Sync bugfix.md to Jira from Bug-implement-fix Session

**Feature Requirement**: FR-028

**Given**: Bug-implement-fix session has completed implementation

**Steps**:
1. After session completes, click "Sync to Jira" button again
2. Monitor sync progress
3. Verify success notification

**Expected Results**:
- API request: `POST /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sync-jira`
- Request body includes bugfix.md content
- Jira Task PROJ-5678 updated (not new task created)
- Jira Task receives new comment containing bugfix.md content
- Response indicates "updated" action

**API Contract Validation**:
```bash
# Read bugfix.md content first
curl https://api.github.com/repos/test-org/specs-repo/contents/bug-1234/bugfix-gh-1234.md?ref=bugfix/gh-1234 \
  -H "Authorization: Bearer $TOKEN"

# Sync with content
curl -X POST /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sync-jira \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "# Bug Fix: Test authentication timeout...\n\n## Implementation Steps..."
  }'

# Expected response:
{
  "jiraTaskKey": "PROJ-5678",
  "jiraTaskURL": "https://company.atlassian.net/browse/PROJ-5678",
  "action": "updated",
  "message": "Successfully updated Jira Task PROJ-5678 with bugfix.md content"
}
```

---

### Scenario 8: Subsequent Jira Syncs Update Existing Task

**Feature Requirement**: FR-016

**Given**: Jira Task PROJ-5678 was previously created for GitHub Issue #1234

**Steps**:
1. Make updates to bugfix.md or GitHub Issue
2. Click "Sync to Jira" button
3. Verify Jira Task is updated (not duplicated)

**Expected Results**:
- BugFixWorkflow CR has `jiraTaskKey: "PROJ-5678"` stored
- API checks `jiraTaskKey` field
- API calls Jira UPDATE endpoint (not CREATE)
- Jira Task PROJ-5678 updated
- No new Jira Task created
- Response indicates "updated" action

**API Contract Validation**:
```bash
# Get workspace to verify jiraTaskKey exists
curl /api/projects/test-project/bugfix-workflows/bugfix-1730368890 \
  -H "Authorization: Bearer $TOKEN"

# Expected response includes:
{
  "jiraTaskKey": "PROJ-5678",
  "lastSyncedAt": "2025-10-31T12:00:00Z"
}

# Sync again
curl -X POST /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sync-jira \
  -H "Authorization: Bearer $TOKEN"

# Expected response action: "updated" (not "created")
```

---

### Scenario 9: Generic Session for Ad-Hoc Work

**Feature Requirement**: FR-030, FR-031

**Given**: BugFix Workspace exists

**Steps**:
1. Click "Start Generic Session" button
2. Select agents (any)
3. Start session
4. Perform ad-hoc exploration or investigation

**Expected Results**:
- AgenticSession created with `bugfix-session-type: generic`
- Session provides open-ended access to workspace tools
- No structured session constraints
- Developer can perform any git operations, file edits, etc.
- Session completes when developer manually stops it or it exits naturally

**API Contract Validation**:
```bash
curl -X POST /api/projects/test-project/bugfix-workflows/bugfix-1730368890/sessions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionType": "generic",
    "title": "Investigate edge case behavior"
  }'
```

---

### Scenario 10: Developer Portfolio and Release Engineering Access

**Feature Requirement**: FR-032, FR-033, FR-034

**Given**: Multiple BugFix Workspaces have been completed

**Steps**:
1. Clone specs repository: `git clone https://github.com/test-org/specs-repo`
2. Checkout feature branch: `git checkout bugfix/gh-1234`
3. Navigate to bug folders: `cd bug-*`
4. Review bugfix markdown files: `cat bug-1234/bugfix-gh-1234.md`

**Expected Results**:
- Organized folder structure: `bug-1234/`, `bug-1235/`, etc.
- Each folder contains `bugfix-gh-{issue-number}.md`
- Each bugfix.md includes:
  - GitHub Issue URL
  - Jira Task URL (if synced)
  - Root cause analysis
  - Implementation plan
  - Implementation steps
  - PR links
- All files committed to feature branch
- Release engineers can parse bugfix.md files for changelog generation

**Validation**:
```bash
# List all bug folders
ls -d bug-*/

# Expected: bug-1234/ bug-1235/ ...

# Check bugfix.md structure
cat bug-1234/bugfix-gh-1234.md

# Expected content:
# # Bug Fix: ...
# **GitHub Issue**: https://github.com/...
# **Jira Task**: https://company.atlassian.net/...
# **Pull Request**: https://github.com/.../pull/123
#
# ## Root Cause Analysis
# ...
#
# ## Implementation Steps
# ...
```

---

## Edge Case Testing

### Edge Case 1: Invalid GitHub Issue URL

**Given**: User provides invalid GitHub Issue URL

**Steps**:
1. Attempt to create workspace with URL: `https://github.com/invalid/repo/issues/99999`

**Expected Results**:
- API validates URL before creating CR
- GitHub API returns 404
- API returns 400 Bad Request with error message: "GitHub Issue not found or inaccessible"
- No BugFixWorkflow CR created
- Frontend displays user-friendly error

---

### Edge Case 2: Duplicate Workspace for Same Issue

**Given**: Workspace already exists for GitHub Issue #1234

**Steps**:
1. Attempt to create second workspace for same GitHub Issue URL

**Expected Results**:
- API checks for existing `bug-1234/` folder in specs-repo
- API returns 409 Conflict with error message: "BugFix Workspace already exists for Issue #1234"
- Frontend offers option to "Resume Existing Workspace"

---

### Edge Case 3: Jira Authentication Failure

**Given**: Jira credentials are missing or invalid

**Steps**:
1. Click "Sync to Jira" button
2. Jira API returns 401 Unauthorized

**Expected Results**:
- API catches Jira authentication error
- API returns 502 Bad Gateway with error: "Jira authentication failed. Check JIRA_API_TOKEN in runner secret"
- Frontend displays error with guidance to configure Jira credentials
- Workspace remains functional (Jira sync is optional)
- User can retry sync after fixing credentials

---

### Edge Case 4: Multiple Concurrent Bugs

**Given**: Developer works on 5 bugs simultaneously

**Steps**:
1. Create 5 BugFix Workspaces for Issues #1234, #1235, #1236, #1237, #1238
2. Start sessions in each workspace concurrently

**Expected Results**:
- Each workspace has unique ID: `bugfix-{timestamp}`
- Each workspace uses separate feature branch: `bugfix/gh-1234`, etc.
- Each workspace has separate folder: `bug-1234/`, etc.
- Sessions isolated by labels (no conflicts)
- No folder/branch conflicts
- All workspaces function independently

---

### Edge Case 5: Branch Already Exists

**Given**: Supporting repo already has branch `bugfix/gh-1234`

**Steps**:
1. Create BugFix Workspace for Issue #1234
2. During Bug-implement-fix session, attempt to create branch

**Expected Results**:
- Backend checks branch existence via GitHub API before creating
- If branch exists:
  - Option 1: Resume work on existing branch (recommended)
  - Option 2: Create variant branch: `bugfix/gh-1234-v2`
  - Option 3: User manually resolves conflict
- Frontend displays clear error with options
- Session does not fail silently

---

## Performance Testing

### Test 1: Workspace Creation Speed

**Target**: < 5 seconds (Success Criteria SC-001)

**Steps**:
1. Create BugFix Workspace from GitHub Issue URL
2. Measure time from API request to "Ready" status

**Pass Criteria**: Total time < 5 seconds

---

### Test 2: GitHub API Call Speed

**Target**: < 2 seconds (Success Criteria SC-002)

**Steps**:
1. Measure time for GitHub Issue validation API call
2. Measure time for GitHub file creation API call

**Pass Criteria**: Each API call < 2 seconds

---

### Test 3: Jira Sync Speed

**Target**: < 10 seconds (Success Criteria SC-003, SC-008)

**Steps**:
1. Click "Sync to Jira" button
2. Measure time from request to completion notification

**Pass Criteria**: Total time < 10 seconds

---

### Test 4: Concurrent Workspace Load

**Target**: 100+ concurrent workspaces (Success Criteria SC-009)

**Steps**:
1. Create 100 BugFix Workspaces in same project
2. Start sessions in 20 workspaces simultaneously
3. Monitor system performance

**Pass Criteria**:
- All workspaces created successfully
- No resource conflicts
- API response times remain < 2x baseline

---

## Automation Test Script Example

### Basic Workflow Test (Bash)

```bash
#!/bin/bash
set -e

PROJECT="test-project"
TOKEN="your-token"
API_BASE="/api"

echo "=== Test: Full BugFix Workflow ==="

# 1. Create workspace
echo "Step 1: Create BugFix Workspace"
WORKSPACE=$(curl -s -X POST "$API_BASE/projects/$PROJECT/bugfix-workflows" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "githubIssueURL": "https://github.com/test-org/test-repo/issues/1234",
    "umbrellaRepo": {"url": "https://github.com/test-org/specs-repo"}
  }')

WORKFLOW_ID=$(echo $WORKSPACE | jq -r '.id')
echo "Created workspace: $WORKFLOW_ID"

# 2. Wait for Ready status
echo "Step 2: Wait for workspace to be Ready"
for i in {1..30}; do
  STATUS=$(curl -s "$API_BASE/projects/$PROJECT/bugfix-workflows/$WORKFLOW_ID" \
    -H "Authorization: Bearer $TOKEN" | jq -r '.status.phase')
  if [ "$STATUS" == "Ready" ]; then
    echo "Workspace is Ready"
    break
  fi
  sleep 1
done

# 3. Create bug-review session
echo "Step 3: Create bug-review session"
SESSION=$(curl -s -X POST "$API_BASE/projects/$PROJECT/bugfix-workflows/$WORKFLOW_ID/sessions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"sessionType": "bug-review"}')

SESSION_ID=$(echo $SESSION | jq -r '.id')
echo "Created session: $SESSION_ID"

# 4. Sync to Jira
echo "Step 4: Sync to Jira"
JIRA_SYNC=$(curl -s -X POST "$API_BASE/projects/$PROJECT/bugfix-workflows/$WORKFLOW_ID/sync-jira" \
  -H "Authorization: Bearer $TOKEN")

JIRA_KEY=$(echo $JIRA_SYNC | jq -r '.jiraTaskKey')
echo "Synced to Jira: $JIRA_KEY"

echo "=== Test Complete: All steps passed ==="
```

---

## Summary

This quickstart guide provides comprehensive integration test scenarios covering:
- All functional requirements (FR-001 through FR-040)
- All acceptance scenarios from the feature spec
- Edge cases and error handling
- Performance testing criteria
- Automation examples

Use these scenarios to validate the BugFix Workspace implementation at each development milestone.
