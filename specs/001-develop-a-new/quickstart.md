# Quickstart: Jira-Session Artifact Integration

**Feature**: Link AgenticSession artifacts to Jira issues
**Date**: 2025-10-09
**Prerequisites**: vTeam deployed, AgenticSession CRD available, Jira instance accessible

---

## Overview
This quickstart validates the Jira integration feature by walking through the complete user workflow from configuration to successful artifact push.

---

## Setup Steps

### 1. Configure Jira Credentials
**Goal**: Store Jira connection details in project runner secrets

```bash
# Set your project namespace and Jira details
export PROJECT_NS="vteam-demo"
export JIRA_URL="https://company.atlassian.net"
export JIRA_PROJECT="DEMO"
export JIRA_API_TOKEN="your-jira-api-token"

# Create or update runner secret
kubectl create secret generic ambient-runner-secrets \
  -n $PROJECT_NS \
  --from-literal=JIRA_URL=$JIRA_URL \
  --from-literal=JIRA_PROJECT=$JIRA_PROJECT \
  --from-literal=JIRA_API_TOKEN=$JIRA_API_TOKEN \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Validation**:
```bash
# Verify secret exists and has required keys
kubectl get secret ambient-runner-secrets -n $PROJECT_NS -o jsonpath='{.data}' | jq 'keys'
# Expected: ["JIRA_API_TOKEN", "JIRA_PROJECT", "JIRA_URL"]
```

---

### 2. Create Test Jira Issue
**Goal**: Prepare a Jira issue to receive artifacts

**Manual Steps**:
1. Log in to your Jira instance
2. Navigate to the project specified in JIRA_PROJECT
3. Create a new issue:
   - Type: Task or Story
   - Summary: "Test vTeam Session Integration"
   - Description: "Testing artifact push from vTeam AgenticSession"
4. Note the issue key (e.g., DEMO-123)

**Validation**:
```bash
# Verify issue is accessible via API
export ISSUE_KEY="DEMO-123"
curl -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/$ISSUE_KEY" | jq '.key, .fields.summary'
# Expected: "DEMO-123" and issue summary
```

---

### 3. Create Test AgenticSession
**Goal**: Run a simple session that generates artifacts

```yaml
# Save as test-session.yaml
apiVersion: vteam.ambient-code/v1alpha1
kind: AgenticSession
metadata:
  name: jira-test-session
  namespace: vteam-demo
spec:
  prompt: "Echo 'Hello from vTeam!' and create a file hello.txt with this message"
  repos:
    - input:
        url: https://github.com/example/test-repo
        branch: main
  llmSettings:
    model: claude-3-5-sonnet-latest
    temperature: 0.7
  timeout: 300
```

```bash
# Create session
kubectl apply -f test-session.yaml

# Wait for completion (max 5 minutes)
kubectl wait --for=condition=complete --timeout=5m \
  agenticsession/jira-test-session -n $PROJECT_NS || true

# Check status
kubectl get agenticsession jira-test-session -n $PROJECT_NS -o jsonpath='{.status.phase}'
# Expected: "Completed" or "Failed"
```

**Validation**:
```bash
# Verify stateDir is populated
STATE_DIR=$(kubectl get agenticsession jira-test-session -n $PROJECT_NS -o jsonpath='{.status.stateDir}')
echo "State directory: $STATE_DIR"
# Should show a path like /vteam/sessions/vteam-demo/jira-test-session
```

---

## Feature Workflow Tests

### Test 1: List Available Artifacts
**Goal**: Retrieve artifact list from session

**API Call**:
```bash
# Get user token (depends on your auth setup)
export USER_TOKEN=$(kubectl create token default -n $PROJECT_NS)

# List artifacts
curl -H "Authorization: Bearer $USER_TOKEN" \
  "http://vteam-backend/api/projects/$PROJECT_NS/sessions/jira-test-session/artifacts" | jq

# Expected response:
# {
#   "artifacts": [
#     {"path": "transcript.txt", "size": 12345, "mimeType": "text/plain"},
#     {"path": "result_summary.txt", "size": 5678, "mimeType": "text/plain"}
#   ]
# }
```

**Acceptance Criteria**:
- ✅ Returns 200 status code
- ✅ `artifacts` array is not empty
- ✅ Each artifact has `path`, `size`, `mimeType` fields
- ✅ All artifact paths are valid filenames (no path traversal)

---

### Test 2: Validate Jira Issue
**Goal**: Check that target Jira issue is accessible

**API Call**:
```bash
curl -X POST \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"issueKey\": \"$ISSUE_KEY\"}" \
  "http://vteam-backend/api/projects/$PROJECT_NS/sessions/jira-test-session/jira/validate" | jq

# Expected response:
# {
#   "valid": true,
#   "issue": {
#     "key": "DEMO-123",
#     "summary": "Test vTeam Session Integration",
#     "status": "To Do",
#     "project": "DEMO"
#   }
# }
```

**Acceptance Criteria**:
- ✅ Returns 200 status code
- ✅ `valid` is `true`
- ✅ `issue` object contains correct key and summary
- ✅ Invalid issue key returns `valid: false` with error message

**Error Case Test**:
```bash
# Test with non-existent issue
curl -X POST \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"issueKey": "DEMO-99999"}' \
  "http://vteam-backend/api/projects/$PROJECT_NS/sessions/jira-test-session/jira/validate" | jq

# Expected: {"valid": false, "error": "Issue not found"}
```

---

### Test 3: Push Artifacts to Jira
**Goal**: Upload session artifacts to Jira issue

**API Call**:
```bash
curl -X POST \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"issueKey\": \"$ISSUE_KEY\",
    \"artifacts\": [\"transcript.txt\", \"result_summary.txt\"]
  }" \
  "http://vteam-backend/api/projects/$PROJECT_NS/sessions/jira-test-session/jira" | jq

# Expected response:
# {
#   "success": true,
#   "jiraKey": "DEMO-123",
#   "attachments": ["transcript.txt", "result_summary.txt"],
#   "commentId": "10001"
# }
```

**Acceptance Criteria**:
- ✅ Returns 200 status code
- ✅ `success` is `true`
- ✅ `attachments` array matches requested artifacts
- ✅ `commentId` is present (comment was created)

---

### Test 4: Verify Jira Issue Updated
**Goal**: Confirm artifacts and comment appear in Jira

**Manual Verification**:
1. Open Jira issue in browser: `$JIRA_URL/browse/$ISSUE_KEY`
2. Check attachments section:
   - ✅ `transcript.txt` is attached
   - ✅ `result_summary.txt` is attached
   - ✅ File sizes match reported sizes
3. Check comments section:
   - ✅ Comment from vTeam bot exists
   - ✅ Comment includes session name, prompt, cost, duration
   - ✅ Comment includes link back to vTeam session

**API Verification**:
```bash
# Check attachments via Jira API
curl -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/$ISSUE_KEY?fields=attachment,comment" | jq '.fields.attachment[].filename'

# Expected output includes:
# "transcript.txt"
# "result_summary.txt"

# Check comment content
curl -H "Authorization: Bearer $JIRA_API_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/$ISSUE_KEY?fields=comment" | jq '.fields.comment.comments[-1].body'

# Expected: Comment text starting with "🤖 vTeam AgenticSession:"
```

---

### Test 5: Verify Session Annotations Updated
**Goal**: Confirm JiraLink stored in AgenticSession metadata

```bash
kubectl get agenticsession jira-test-session -n $PROJECT_NS \
  -o jsonpath='{.metadata.annotations.vteam\.ambient-code/jiraLinks}' | jq

# Expected output:
# [
#   {
#     "path": "transcript.txt",
#     "jiraKey": "DEMO-123",
#     "timestamp": "2025-10-09T10:00:00Z",
#     "status": "success"
#   },
#   {
#     "path": "result_summary.txt",
#     "jiraKey": "DEMO-123",
#     "timestamp": "2025-10-09T10:00:00Z",
#     "status": "success"
#   }
# ]
```

**Acceptance Criteria**:
- ✅ `jiraLinks` annotation exists
- ✅ Contains entry for each uploaded artifact
- ✅ All entries have `status: "success"`
- ✅ Timestamp is recent (within last hour)

---

### Test 6: Retrieve Jira Links via API
**Goal**: Read back Jira links from session

```bash
curl -H "Authorization: Bearer $USER_TOKEN" \
  "http://vteam-backend/api/projects/$PROJECT_NS/sessions/jira-test-session/jira" | jq

# Expected response:
# {
#   "links": [
#     {
#       "path": "transcript.txt",
#       "jiraKey": "DEMO-123",
#       "timestamp": "2025-10-09T10:00:00Z",
#       "status": "success"
#     },
#     ...
#   ]
# }
```

**Acceptance Criteria**:
- ✅ Returns 200 status code
- ✅ `links` array matches annotation content
- ✅ Links are sorted by timestamp (most recent first)

---

## Error Scenario Tests

### Test 7: Missing Jira Configuration
**Goal**: Verify graceful error when credentials not configured

```bash
# Create namespace without Jira secret
kubectl create namespace vteam-no-jira
kubectl apply -f test-session.yaml -n vteam-no-jira

# Wait for session completion, then try push
curl -X POST \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"issueKey\": \"$ISSUE_KEY\", \"artifacts\": [\"transcript.txt\"]}" \
  "http://vteam-backend/api/projects/vteam-no-jira/sessions/jira-test-session/jira" | jq

# Expected response:
# {
#   "error": "Missing Jira configuration in runner secret (JIRA_URL, JIRA_PROJECT, JIRA_API_TOKEN required)",
#   "code": "JIRA_CONFIG_MISSING",
#   "retryable": false
# }
```

**Acceptance Criteria**:
- ✅ Returns 400 status code
- ✅ Error message mentions missing configuration
- ✅ Suggests checking Project Settings

---

### Test 8: Artifact Too Large
**Goal**: Handle files exceeding Jira's 10MB limit

```bash
# Create large test file in session stateDir (requires backend implementation or mock)
# Attempt push with large artifact
curl -X POST \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"issueKey\": \"$ISSUE_KEY\", \"artifacts\": [\"large_file.log\"]}" \
  "http://vteam-backend/api/projects/$PROJECT_NS/sessions/jira-test-session/jira" | jq

# Expected response:
# {
#   "success": false,
#   "jiraKey": "DEMO-123",
#   "attachments": [],
#   "errors": [
#     {"path": "large_file.log", "error": "File exceeds 10MB limit"}
#   ]
# }
```

**Acceptance Criteria**:
- ✅ Returns 200 status code (partial failure)
- ✅ `success` is `false`
- ✅ `errors` array explains size issue
- ✅ No attachment created in Jira

---

### Test 9: Invalid Jira Token
**Goal**: Handle authentication failure gracefully

```bash
# Temporarily update secret with invalid token
kubectl create secret generic ambient-runner-secrets \
  -n $PROJECT_NS \
  --from-literal=JIRA_URL=$JIRA_URL \
  --from-literal=JIRA_PROJECT=$JIRA_PROJECT \
  --from-literal=JIRA_API_TOKEN="invalid-token" \
  --dry-run=client -o yaml | kubectl apply -f -

# Attempt push
curl -X POST \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"issueKey\": \"$ISSUE_KEY\", \"artifacts\": [\"transcript.txt\"]}" \
  "http://vteam-backend/api/projects/$PROJECT_NS/sessions/jira-test-session/jira" | jq

# Expected response:
# {
#   "error": "Jira authentication failed",
#   "code": "JIRA_AUTH_FAILED",
#   "details": "Invalid API token",
#   "retryable": false
# }

# Restore valid token
kubectl create secret generic ambient-runner-secrets \
  -n $PROJECT_NS \
  --from-literal=JIRA_URL=$JIRA_URL \
  --from-literal=JIRA_PROJECT=$JIRA_PROJECT \
  --from-literal=JIRA_API_TOKEN=$JIRA_API_TOKEN \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Acceptance Criteria**:
- ✅ Returns 401 status code
- ✅ Error message mentions authentication failure
- ✅ `retryable` is `false`

---

## Frontend UI Tests (Manual)

### Test 10: Push Dialog Workflow
**Goal**: Validate end-to-end UI interaction

**Steps**:
1. Navigate to session detail page: `/projects/$PROJECT_NS/sessions/jira-test-session`
2. Verify "Push to Jira" button is visible and enabled
3. Click "Push to Jira" button
4. Dialog opens with:
   - ✅ Issue key input field
   - ✅ Artifact checklist (all checked by default)
   - ✅ "Validate Issue" button
   - ✅ "Push" button (disabled initially)
5. Enter issue key: `DEMO-123`
6. Click "Validate Issue"
7. Verify:
   - ✅ Issue metadata displayed (title, status)
   - ✅ "Push" button now enabled
8. Click "Push"
9. Verify:
   - ✅ Progress indicator appears
   - ✅ Success message displayed with Jira link
   - ✅ Dialog closes
10. Verify session page now shows:
    - ✅ "Linked to Jira: DEMO-123" badge
    - ✅ Badge is clickable (opens Jira in new tab)

---

### Test 11: Push History View
**Goal**: Verify push history display

**Steps**:
1. Push artifacts to 2 different Jira issues (DEMO-123, DEMO-456)
2. Navigate to session detail page
3. Expand "Push History" section
4. Verify table displays:
   - ✅ 2 rows (one per issue)
   - ✅ Columns: Issue Key, Artifacts, Timestamp, Status
   - ✅ All statuses are "Success"
   - ✅ Timestamps are sorted (most recent first)

---

## Cleanup

```bash
# Delete test session
kubectl delete agenticsession jira-test-session -n $PROJECT_NS

# Delete test namespace (if created)
kubectl delete namespace vteam-no-jira

# Optionally delete Jira issue (via Jira UI)
```

---

## Success Criteria Summary

This quickstart validates that:
1. ✅ Jira configuration is stored and loaded correctly
2. ✅ Session artifacts are discovered and listed
3. ✅ Jira issue validation works for valid/invalid keys
4. ✅ Artifacts upload to Jira successfully
5. ✅ Jira comment is created with session metadata
6. ✅ Session annotations are updated with JiraLinks
7. ✅ Error handling works for common failures
8. ✅ UI provides clear workflow and feedback
9. ✅ Push history is tracked and displayed

**Overall**: Feature is production-ready when all acceptance criteria pass.

---

## Troubleshooting

### Issue: Backend returns 401 Unauthorized
**Solution**: Verify user token is valid with `kubectl auth can-i get agenticsessions -n $PROJECT_NS`

### Issue: Jira validation fails with network error
**Solution**: Check backend can reach Jira URL. Test with `curl $JIRA_URL` from backend pod.

### Issue: Attachments don't appear in Jira
**Solution**:
1. Check Jira API response in backend logs
2. Verify JIRA_API_TOKEN has "Add Attachments" permission
3. Confirm artifact files exist in stateDir

### Issue: Session doesn't have stateDir in status
**Solution**: Wait for session to reach Running/Completed phase. StateDir is populated by runner after pod starts.
