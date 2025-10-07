# Data Model: Jira Integration

**Feature**: 001-jira-integration
**Date**: 2025-10-07

## Entity: JiraConfiguration

**Description**: Stores organization or user-level Jira integration settings including OAuth credentials and publishing preferences.

**Attributes**:
- `id` (string, UUID): Unique identifier for configuration
- `organization_id` (string, UUID): Organization owning this configuration (nullable for user-level configs)
- `user_id` (string, UUID): User owning this configuration (nullable for org-level configs)
- `client_id` (string): OAuth 2.0 client ID from Atlassian Developer Console
- `client_secret` (string, encrypted): OAuth 2.0 client secret (encrypted at rest)
- `access_token` (string, encrypted): Current OAuth access token (encrypted at rest)
- `refresh_token` (string, encrypted): Current OAuth refresh token (encrypted at rest)
- `token_expires_at` (timestamp): When access token expires
- `token_refreshed_at` (timestamp): Last successful token refresh
- `atlassian_cloud_id` (string): Atlassian Cloud site ID
- `default_project_key` (string): Default Jira project key (optional)
- `publishing_strategy` (enum): "attachments", "comments", "description", "attachments_and_comments"
- `enabled` (boolean): Whether integration is active
- `created_at` (timestamp): When configuration was created
- `updated_at` (timestamp): Last configuration update

**Relationships**:
- One JiraConfiguration per organization or user
- JiraConfiguration has many PublishingOperations

**Validation Rules**:
- `client_id` must not be empty if `enabled` is true
- `client_secret` must not be empty if `enabled` is true
- `publishing_strategy` must be one of allowed values
- Either `organization_id` or `user_id` must be set, not both
- `token_expires_at` must be future timestamp

**State Transitions**:
- `enabled: false → true`: Requires valid OAuth credentials
- `enabled: true → false`: Clears tokens (keeps client_id/secret)
- Token refresh: Updates `access_token`, `refresh_token`, `token_expires_at`, `token_refreshed_at`

---

## Entity: SessionJiraLink

**Description**: Links a vTeam workflow session to a Jira issue key.

**Attributes**:
- `id` (string, UUID): Unique identifier
- `session_id` (string, UUID): vTeam session identifier
- `issue_key` (string): Jira issue key (format: PROJECT-123)
- `issue_id` (string): Jira internal issue ID (fetched on first publish)
- `issue_summary` (string): Cached Jira issue summary
- `linked_at` (timestamp): When link was created
- `linked_by_user_id` (string, UUID): User who created link
- `publishing_status` (enum): "pending", "in_progress", "completed", "failed", "retry_queued"
- `last_published_at` (timestamp, nullable): Last successful publish
- `last_error` (string, nullable): Last error message if failed
- `retry_count` (integer): Number of retry attempts
- `next_retry_at` (timestamp, nullable): When next retry is scheduled

**Relationships**:
- Many SessionJiraLinks per session (if re-linked)
- One active SessionJiraLink per session (publishing_status != failed)
- SessionJiraLink has many PublishingOperations
- SessionJiraLink belongs to one JiraConfiguration (via organization/user)

**Validation Rules**:
- `issue_key` must match pattern: `^[A-Z][A-Z0-9]*-[0-9]+$`
- `session_id` must reference valid vTeam session
- `publishing_status` must be one of allowed values
- `retry_count` must be >= 0 and <= 10 (max retries)
- Only one link per session can have `publishing_status` in ["pending", "in_progress", "completed", "retry_queued"]

**State Transitions**:
```
pending → in_progress: Publishing started
in_progress → completed: All artifacts published successfully
in_progress → failed: Unrecoverable error (4xx except 401, 429)
in_progress → retry_queued: Transient error (401, 429, 5xx, network)
retry_queued → in_progress: Retry attempt started
failed → pending: User manually retries
```

---

## Entity: Artifact

**Description**: Represents a generated artifact file (spec.md, tasks.md, plan.md) from a session.

**Attributes**:
- `id` (string, UUID): Unique identifier
- `session_id` (string, UUID): Session that generated artifact
- `type` (enum): "spec", "tasks", "plan"
- `file_path` (string): Absolute path to artifact file
- `file_size` (integer): Size in bytes
- `content_hash` (string): SHA-256 hash of content (for change detection)
- `generated_at` (timestamp): When artifact was generated
- `generated_by` (string): Agent or user that generated artifact

**Relationships**:
- Artifact belongs to one Session
- Artifact has many ArtifactPublications

**Validation Rules**:
- `type` must be one of: "spec", "tasks", "plan"
- `file_path` must exist and be readable
- `file_size` must be > 0 and <= 10MB (warning threshold)
- `content_hash` must be valid SHA-256 hex string

**State Transitions**:
- Created: When session generates artifact
- Updated: If artifact regenerated (new content_hash)

---

## Entity: PublishingOperation

**Description**: Represents a single attempt to publish artifacts to Jira, including detailed status and error information.

**Attributes**:
- `id` (string, UUID): Unique identifier
- `session_jira_link_id` (string, UUID): Link this operation belongs to
- `operation_type` (enum): "full", "attachments_only", "comments_only", "retry"
- `started_at` (timestamp): When operation started
- `completed_at` (timestamp, nullable): When operation completed
- `status` (enum): "in_progress", "completed", "failed"
- `artifacts_published` (array of strings): List of artifact IDs successfully published
- `attachments_created` (array of objects): Jira attachment IDs and filenames
- `comment_id` (string, nullable): Jira comment ID if comment was added
- `error_code` (string, nullable): Error code if failed
- `error_message` (string, nullable): User-friendly error message
- `error_details` (string, nullable): Technical error details (for logging)
- `http_status_code` (integer, nullable): HTTP status from Jira API
- `retry_after_seconds` (integer, nullable): Seconds to wait before retry (from Retry-After header)
- `duration_ms` (integer): Operation duration in milliseconds

**Relationships**:
- PublishingOperation belongs to one SessionJiraLink
- PublishingOperation references many Artifacts

**Validation Rules**:
- `operation_type` must be one of allowed values
- `status` must be one of allowed values
- `started_at` must be <= `completed_at` if `completed_at` is set
- `http_status_code` must be valid HTTP status (100-599)
- `duration_ms` must be > 0 if `completed_at` is set

**State Transitions**:
```
in_progress → completed: All artifacts published
in_progress → failed: Error occurred
```

---

## Entity: JiraPublishQueue

**Description**: Background queue for retrying failed publishing operations.

**Attributes**:
- `id` (string, UUID): Unique identifier
- `session_jira_link_id` (string, UUID): Link to retry
- `scheduled_at` (timestamp): When retry is scheduled
- `attempts` (integer): Number of attempts so far
- `last_error` (string): Last error message
- `created_at` (timestamp): When queued
- `status` (enum): "queued", "processing", "completed", "abandoned"

**Relationships**:
- JiraPublishQueue belongs to one SessionJiraLink

**Validation Rules**:
- `attempts` must be >= 0 and <= 10
- `scheduled_at` must be future timestamp
- `status` must be one of allowed values

**State Transitions**:
```
queued → processing: Retry attempt started
processing → completed: Retry succeeded
processing → queued: Retry failed, schedule next attempt
processing → abandoned: Max attempts exceeded
```

---

## Enumerations

### PublishingStatus
- `pending`: Waiting to publish
- `in_progress`: Currently publishing
- `completed`: Successfully published
- `failed`: Failed with unrecoverable error
- `retry_queued`: Queued for automatic retry

### PublishingStrategy
- `attachments`: Upload artifacts as attachments only
- `comments`: Add comment with summary only
- `description`: Update issue description (risky, not recommended)
- `attachments_and_comments`: Upload attachments + add comment (recommended)

### ArtifactType
- `spec`: Feature specification (spec.md)
- `tasks`: Task breakdown (tasks.md)
- `plan`: Implementation plan (plan.md)

### OperationType
- `full`: Publish all artifacts with configured strategy
- `attachments_only`: Upload attachments only (fallback)
- `comments_only`: Add comment only (if attachments already exist)
- `retry`: Retry previous failed operation

### QueueStatus
- `queued`: Waiting for retry
- `processing`: Currently retrying
- `completed`: Retry succeeded
- `abandoned`: Exceeded max retries

---

## Data Relationships Diagram

```
JiraConfiguration
    |
    | (1:many)
    |
SessionJiraLink
    |
    | (1:many)
    |
PublishingOperation
    |
    | (many:many)
    |
Artifact

SessionJiraLink
    |
    | (1:1)
    |
JiraPublishQueue
```

---

## Storage Considerations

### Security:
- `client_secret`, `access_token`, `refresh_token` must be encrypted at rest
- Use AES-256 encryption with key from secrets manager
- Never log sensitive fields
- Rotate encryption keys periodically

### Performance:
- Index on `session_id` for SessionJiraLink queries
- Index on `publishing_status` for retry queue processing
- Index on `scheduled_at` for JiraPublishQueue background jobs
- Index on `organization_id` and `user_id` for JiraConfiguration lookups

### Retention:
- PublishingOperation records: Keep for 90 days (audit trail)
- JiraPublishQueue records: Delete when status is "completed" or "abandoned"
- SessionJiraLink records: Keep indefinitely (linked to session lifecycle)
- JiraConfiguration records: Keep while `enabled` or has recent activity

---

## Data Migration Considerations

If vTeam previously had custom Jira integration:
1. Map old configuration to new JiraConfiguration entity
2. Migrate OAuth credentials (re-encrypt if needed)
3. Preserve historical publishing status if available
4. Update foreign keys for session references

---

## Validation Examples

### Valid Issue Key:
- ✅ `PROJ-123`
- ✅ `ABC-1`
- ✅ `PROJECT2-9999`
- ❌ `proj-123` (lowercase)
- ❌ `PROJ123` (missing hyphen)
- ❌ `PROJ-` (missing number)

### Valid Publishing Status Transitions:
```
pending → in_progress ✅
in_progress → completed ✅
in_progress → failed ✅
in_progress → retry_queued ✅
failed → pending ✅ (manual retry)
completed → pending ❌ (cannot republish completed)
retry_queued → in_progress ✅
```
