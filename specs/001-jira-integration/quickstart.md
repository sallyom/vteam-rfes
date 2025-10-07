# Quickstart: Jira Integration

**Feature**: 001-jira-integration
**Audience**: End users, QA testers
**Estimated Time**: 15 minutes

## Overview

This quickstart validates the end-to-end Jira integration workflow:
1. Configure OAuth credentials
2. Link a session to a Jira issue
3. Publish artifacts automatically on session completion
4. Verify artifacts appear in Jira

## Prerequisites

- [ ] vTeam backend and frontend running locally or in test environment
- [ ] Access to Atlassian Cloud (Jira) with admin permissions
- [ ] Ability to create OAuth 2.0 apps in Atlassian Developer Console
- [ ] Test Jira project (e.g., "TEST") with permission to create/edit issues

## Step 1: Create OAuth App in Atlassian Developer Console

1. Navigate to https://developer.atlassian.com/console/myapps/
2. Click **Create** > **OAuth 2.0 integration**
3. Enter app details:
   - **App name**: `vTeam Integration (Test)`
   - **Description**: `OAuth app for vTeam artifact publishing`
4. Click **Save**
5. Navigate to **Permissions** tab:
   - Add `write:jira-work` scope
   - Add `read:jira-work` scope
   - Add `read:me` scope
6. Navigate to **Authorization** tab:
   - Add callback URL: `http://localhost:3000/api/v1/jira/oauth/callback`
   - (Replace with your actual callback URL)
7. Navigate to **Settings** tab:
   - Copy **Client ID**
   - Copy **Client Secret** (will only be shown once)

**Expected Result**: OAuth app created with client ID and secret

## Step 2: Configure Jira Integration in vTeam

### Via UI:

1. Open vTeam in browser: `http://localhost:3000`
2. Navigate to **Settings** > **Integrations** > **Jira**
3. Click **Configure Jira Integration**
4. Enter OAuth credentials:
   - **Client ID**: `<your-client-id>`
   - **Client Secret**: `<your-client-secret>`
5. Select **Publishing Strategy**: `Attachments and Comments`
6. Click **Authorize with Atlassian**
7. Redirected to Atlassian authorization page
8. Click **Accept** to grant permissions
9. Redirected back to vTeam with success message

**Expected Result**: Integration status shows "Connected" with green indicator

### Via API (Alternative):

```bash
# 1. Create configuration
curl -X POST http://localhost:8080/api/v1/jira/config \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": "YOUR_CLIENT_SECRET",
    "publishing_strategy": "attachments_and_comments",
    "enabled": true
  }'

# 2. Get authorization URL
curl -X GET "http://localhost:8080/api/v1/jira/oauth/authorize?redirect_uri=http://localhost:3000/oauth-callback" \
  -H "Authorization: Bearer $JWT_TOKEN"

# Response includes authorization_url and state
# Visit authorization_url in browser, grant permissions

# 3. Exchange code for tokens (after redirect)
curl -X POST http://localhost:8080/api/v1/jira/oauth/callback \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "AUTH_CODE_FROM_REDIRECT",
    "state": "STATE_FROM_STEP_2"
  }'
```

**Expected Result**: API returns configuration with `token_expires_at` set

## Step 3: Create Test Jira Issue

1. Navigate to your Jira test project
2. Click **Create** issue
3. Enter details:
   - **Project**: TEST
   - **Issue Type**: Story
   - **Summary**: "Test vTeam Integration - Artifact Publishing"
   - **Description**: "This issue is for testing automated artifact publishing from vTeam"
4. Click **Create**
5. Note the **Issue Key** (e.g., `TEST-123`)

**Expected Result**: Jira issue created with key `TEST-123`

## Step 4: Create and Link vTeam Session

### Via UI:

1. In vTeam, create a new workflow session:
   - Click **New Session**
   - Enter feature description: "Dark mode toggle for settings"
2. In session details panel, find **Jira Integration** section
3. Click **Link to Jira Issue**
4. Enter issue key: `TEST-123`
5. Click **Link**
6. Verify issue summary appears: "Test vTeam Integration - Artifact Publishing"

**Expected Result**: Session shows "Linked to TEST-123" with issue summary

### Via API (Alternative):

```bash
SESSION_ID="your-session-uuid"

curl -X POST http://localhost:8080/api/v1/sessions/$SESSION_ID/jira-link \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "issue_key": "TEST-123"
  }'
```

**Expected Result**: API returns SessionJiraLink with `publishing_status: "pending"`

## Step 5: Generate Session Artifacts

1. In the vTeam session, run workflows to generate artifacts:
   - Run `/specify` to create `spec.md`
   - Run `/plan` to create `plan.md` and `research.md`
   - Run `/tasks` to create `tasks.md`

2. Verify artifacts exist in session:
   ```bash
   ls specs/001-dark-mode/
   # Should show: spec.md, plan.md, research.md, tasks.md
   ```

**Expected Result**: Three artifacts generated (spec.md, tasks.md, plan.md)

## Step 6: Complete Session and Trigger Publishing

### Automatic Publishing (Recommended):

1. Mark session as complete:
   - Click **Complete Session** button in UI
   - Or via API:
     ```bash
     curl -X POST http://localhost:8080/api/v1/sessions/$SESSION_ID/complete \
       -H "Authorization: Bearer $JWT_TOKEN"
     ```

2. Publishing should start automatically
3. Watch for notification: "Publishing artifacts to TEST-123..."

**Expected Result**: Notification shows "Successfully published artifacts to TEST-123"

### Manual Publishing (Alternative):

```bash
curl -X POST http://localhost:8080/api/v1/sessions/$SESSION_ID/publish \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "artifacts": ["spec", "tasks", "plan"]
  }'
```

**Expected Result**: API returns `202 Accepted` with PublishingOperation in `in_progress` status

## Step 7: Verify Artifacts in Jira

1. Open Jira issue `TEST-123` in browser
2. Check **Attachments** section:
   - Should see `spec-{session-id}.md`
   - Should see `tasks-{session-id}.md`
   - Should see `plan-{session-id}.md`
3. Check **Comments**:
   - Should see comment from vTeam integration
   - Comment should list published artifacts with session ID
4. Download and verify `spec-{session-id}.md`:
   - Click attachment to download
   - Open in text editor
   - Verify markdown formatting is preserved
   - Content matches generated spec.md

**Expected Result**: All three artifacts attached, comment added, content readable

## Step 8: Check Publishing Status

### Via UI:

1. In vTeam session, navigate to **Publishing Status** tab
2. Verify status shows:
   - Publishing Status: **Completed**
   - Last Published: `<timestamp>`
   - Artifacts Published: 3
   - Attachments Created: 3
   - Comment ID: `<jira-comment-id>`

### Via API:

```bash
curl -X GET http://localhost:8080/api/v1/sessions/$SESSION_ID/publish/status \
  -H "Authorization: Bearer $JWT_TOKEN"
```

**Expected Response**:
```json
{
  "link": {
    "id": "...",
    "session_id": "...",
    "issue_key": "TEST-123",
    "publishing_status": "completed",
    "last_published_at": "2025-10-07T10:30:00Z",
    "retry_count": 0
  },
  "operations": [
    {
      "id": "...",
      "operation_type": "full",
      "status": "completed",
      "artifacts_published": ["spec-id", "tasks-id", "plan-id"],
      "attachments_created": [
        {"attachment_id": "10000", "filename": "spec-abc123.md"},
        {"attachment_id": "10001", "filename": "tasks-abc123.md"},
        {"attachment_id": "10002", "filename": "plan-abc123.md"}
      ],
      "comment_id": "10005",
      "duration_ms": 2340
    }
  ]
}
```

## Step 9: Test Error Handling - Invalid Issue Key

1. Create a new session in vTeam
2. Link to invalid issue: `TEST-99999` (doesn't exist)
3. Complete session and trigger publishing
4. Verify error handling:
   - Notification: "Failed to publish to TEST-99999: Issue not found"
   - Publishing status: **Failed**
   - Error message includes: "Issue may not exist or you don't have permission"
   - Retry option available

**Expected Result**: Clear error message, publishing marked as failed, retry available

## Step 10: Test Retry Mechanism

### Via UI:

1. Using failed session from Step 9, click **Retry Publishing**
2. Correct issue key to valid issue (e.g., `TEST-123`)
3. Click **Update and Retry**
4. Verify publishing succeeds

### Via API:

```bash
curl -X POST http://localhost:8080/api/v1/sessions/$SESSION_ID/publish/retry \
  -H "Authorization: Bearer $JWT_TOKEN"
```

**Expected Result**: Retry succeeds, artifacts published to corrected issue

## Validation Checklist

After completing the quickstart, verify:

- [ ] OAuth configuration successful with valid tokens
- [ ] Session linked to Jira issue
- [ ] Session completion triggers automatic publishing
- [ ] Three artifacts uploaded as attachments
- [ ] Comment added to Jira issue
- [ ] Markdown formatting preserved in attachments
- [ ] Publishing status shows "completed" with timestamps
- [ ] Invalid issue key produces clear error message
- [ ] Retry mechanism works after correcting error
- [ ] Integration health check shows "healthy" status

## Health Check

Run integration health check:

```bash
curl -X GET http://localhost:8080/api/v1/jira/health \
  -H "Authorization: Bearer $JWT_TOKEN"
```

**Expected Response**:
```json
{
  "status": "healthy",
  "configured": true,
  "token_valid": true,
  "jira_accessible": true,
  "last_successful_publish": "2025-10-07T10:30:00Z",
  "error": null
}
```

## Troubleshooting

### OAuth Authorization Fails

**Symptom**: Redirect to Atlassian returns error "invalid_client"

**Solution**:
- Verify client ID and secret are correct
- Check callback URL matches exactly (including http/https, port)
- Ensure scopes are configured in Atlassian Developer Console

### Publishing Returns 401 Unauthorized

**Symptom**: Publishing fails with "Authentication expired"

**Solution**:
- Token refresh should happen automatically
- If persists, re-authorize via Settings > Integrations > Jira
- Check logs for token refresh errors

### Publishing Returns 403 Forbidden

**Symptom**: Publishing fails with "Insufficient permissions"

**Solution**:
- Verify Jira user has permissions: Browse Projects, Edit Issues, Create Attachments
- Check issue-level security restrictions
- Try with Jira admin account to isolate permission issue

### Artifacts Not Appearing in Jira

**Symptom**: Publishing shows "completed" but no attachments in Jira

**Solution**:
- Verify attachment IDs in PublishingOperation response
- Check Jira issue activity log for attachment events
- Confirm file paths exist and are readable
- Check file size is under 10MB

### Rate Limit Errors

**Symptom**: Publishing fails with "Rate limit exceeded"

**Solution**:
- Wait for automatic retry (exponential backoff)
- Check Publishing Status for next retry time
- If testing with many operations, spread them over time

## Next Steps

After validating the quickstart:

1. **Run Integration Tests**: Execute automated test suite
2. **Test with Real Sessions**: Use actual workflow sessions with larger artifacts
3. **Test Bulk Publishing**: Link and publish multiple sessions
4. **Monitor Performance**: Check dashboard for publishing metrics
5. **Review Audit Logs**: Verify all operations are logged

## Cleanup

To clean up test data:

1. Delete test OAuth app in Atlassian Developer Console
2. Unlink session from Jira issue (does not delete attachments)
3. Optionally delete TEST-123 issue in Jira
4. Delete or disable Jira configuration in vTeam settings

```bash
# Via API
curl -X DELETE http://localhost:8080/api/v1/jira/config \
  -H "Authorization: Bearer $JWT_TOKEN"
```
