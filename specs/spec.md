# Feature Specification: Jira Integration Restoration for vTeam Platform

**Feature Branch**: `jira-integration-restoration`
**Created**: 2025-11-11
**Status**: Draft
**Input**: User description: "Restore Jira integration for the vTeam platform. Allow users to upload individual files from the Shared Artifacts directory as Feature Specs to Jira issues. Support viewing synced files in Jira. The existing Jira integration code has been commented out but the infrastructure remains."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Upload Single Artifact to New Jira Issue (Priority: P1)

As a product manager, I want to upload a specification document from my session's Shared Artifacts directory to create a new Jira issue, so that I can track feature development in Jira without manual copy-paste.

**Why this priority**: This is the core MVP functionality that delivers immediate value. Users can begin syncing their work to Jira with minimal friction. This story is the foundation for all other Jira integration features.

**Independent Test**: Can be fully tested by uploading a single artifact file (e.g., spec.md) from Shared Artifacts and verifying a new Jira issue is created with the file content as the description. Success is measurable by checking the Jira issue exists and contains the correct content.

**Acceptance Scenarios**:

1. **Given** a user has a session with artifacts in the Shared Artifacts directory, **When** they right-click on a file and select "Upload to Jira" > "Create New Issue", **Then** a dialog opens allowing them to configure issue details (summary, issue type, labels)
2. **Given** the upload dialog is open, **When** the user fills in required fields and clicks "Create Issue", **Then** a new Jira issue is created with the file content, and the UI shows a success message
3. **Given** an artifact has been successfully uploaded, **When** the user views the Shared Artifacts directory, **Then** the file displays a Jira badge with the issue key (e.g., "PROJ-123")
4. **Given** the Jira credentials are not configured, **When** the user attempts to upload, **Then** the system displays an error message directing them to configure Jira in Project Settings
5. **Given** the file content contains markdown, **When** uploaded to Jira, **Then** the content is converted to Jira wiki/ADF format appropriately

---

### User Story 2 - View Jira Issue from Artifact (Priority: P1)

As a developer, I want to click on a Jira badge next to a synced artifact file to open the corresponding Jira issue in a new browser tab, so that I can quickly navigate between vTeam and Jira.

**Why this priority**: This completes the basic user journey by providing bidirectional navigation. Without this, users would need to manually search for issues in Jira, defeating the purpose of integration. This is essential for the MVP to be truly useful.

**Independent Test**: Can be tested independently by syncing an artifact (using Story 1), then clicking the Jira badge and verifying it opens the correct Jira issue URL in a new tab. No other functionality is required.

**Acceptance Scenarios**:

1. **Given** an artifact file has been synced to Jira, **When** the user hovers over the Jira badge, **Then** a tooltip appears showing the issue key, summary, and status
2. **Given** the Jira badge is displayed, **When** the user clicks on it, **Then** the corresponding Jira issue opens in a new browser tab
3. **Given** the user right-clicks on a synced file, **When** they view the context menu, **Then** options include "View in Jira" and "Copy Jira Link"
4. **Given** the user selects "Copy Jira Link", **When** the action completes, **Then** the Jira issue URL is copied to clipboard and a confirmation toast appears

---

### User Story 3 - Update Existing Jira Issue (Priority: P2)

As a product manager, I want to update an existing Jira issue when I modify an artifact file that was previously synced, so that Jira stays in sync with my latest changes without creating duplicate issues.

**Why this priority**: This prevents issue proliferation and maintains a single source of truth. While important, users can still derive value from P1 stories alone. This story enables iterative workflows where specs evolve over time.

**Independent Test**: Can be tested by first syncing a file (creating issue), modifying the file content, then re-uploading and choosing "Update Existing Issue". Verify the Jira issue is updated without creating a new issue.

**Acceptance Scenarios**:

1. **Given** an artifact file has been previously synced to Jira, **When** the user right-clicks and selects "Upload to Jira" > "Update Existing Issue", **Then** the system uses the stored Jira link to update the existing issue
2. **Given** the user modifies a file and re-uploads, **When** the update completes, **Then** the Jira issue description is updated with the new content and a comment is added noting the update timestamp
3. **Given** a file was synced but the Jira issue was deleted externally, **When** the user attempts to update, **Then** the system detects the 404 error and offers to create a new issue or unlink the file
4. **Given** the user wants to change which Jira issue a file is linked to, **When** they select "Link to Different Issue", **Then** they can enter a Jira issue key to update the mapping

---

### User Story 4 - Jira Configuration and Validation (Priority: P2)

As a project administrator, I want to configure Jira connection settings in the Project Settings page and test the connection, so that I can ensure the integration is properly configured before team members start using it.

**Why this priority**: This is required infrastructure but doesn't directly deliver user value for uploading/viewing. The frontend configuration UI already exists, so this story focuses on backend validation and credentials storage. Can be delivered after P1 to enable self-service setup.

**Independent Test**: Can be tested independently by entering Jira credentials in Project Settings, clicking "Test Connection", and verifying the connection status. No artifact operations are required.

**Acceptance Scenarios**:

1. **Given** a user navigates to Project Settings > Integrations, **When** they view the Jira section, **Then** fields are displayed for Jira URL, Project Key, Email, and API Token
2. **Given** the user enters valid credentials, **When** they click "Test Connection", **Then** the system verifies connectivity and displays "Connected" status with the Jira project name
3. **Given** the user enters invalid credentials, **When** they click "Test Connection", **Then** the system displays a specific error message (e.g., "Authentication failed", "Project not found")
4. **Given** valid credentials are saved, **When** the user navigates away and returns, **Then** the credentials are persisted (with API token masked) and the connection status is shown
5. **Given** Jira credentials are stored, **When** accessed by the backend, **Then** they are retrieved securely from Kubernetes secrets with appropriate namespacing

---

### User Story 5 - Sync Status Persistence Across Sessions (Priority: P3)

As a team member, I want to see which artifacts have been synced to Jira even after closing and reopening a session, so that I don't accidentally create duplicate issues or lose track of synced content.

**Why this priority**: This improves user experience by providing state persistence, but the core functionality works without it (users can still see sync status during an active session). This is a quality-of-life enhancement rather than core functionality.

**Independent Test**: Can be tested by syncing artifacts to Jira, noting the badges displayed, closing the session, reopening it, and verifying the Jira badges are still displayed with correct issue keys.

**Acceptance Scenarios**:

1. **Given** a user has synced several artifacts to Jira, **When** they close the browser tab and reopen the session later, **Then** all Jira badges are displayed with the correct issue keys
2. **Given** Jira link metadata is stored, **When** the session data is retrieved, **Then** the `jiraLinks` field contains mappings of artifact paths to Jira issue keys
3. **Given** multiple users access the same session, **When** one user syncs an artifact, **Then** other users see the Jira badge after refreshing the workspace
4. **Given** a session is cloned or recreated, **When** artifacts are copied, **Then** the Jira links are preserved in the new session

---

### User Story 6 - Bulk Upload Multiple Artifacts (Priority: P3)

As a product manager, I want to select multiple artifact files and upload them to Jira in a single operation, so that I can efficiently sync an entire feature specification set without uploading files one by one.

**Why this priority**: This is a convenience feature that improves efficiency for power users. The core value is already delivered by single-file uploads (P1). This is an optimization for users with many artifacts.

**Independent Test**: Can be tested independently by selecting multiple files using checkboxes in the Shared Artifacts view, clicking "Upload Selected to Jira", and verifying that multiple Jira issues are created (or a batch operation completes).

**Acceptance Scenarios**:

1. **Given** the user is viewing Shared Artifacts, **When** they click a "Select Multiple" button, **Then** checkboxes appear next to each file
2. **Given** multiple files are selected, **When** the user clicks "Upload to Jira", **Then** a batch upload dialog opens showing all selected files
3. **Given** the batch upload dialog is open, **When** the user configures settings (e.g., common labels, issue type), **Then** these settings apply to all files in the batch
4. **Given** the user confirms the batch upload, **When** processing begins, **Then** a progress indicator shows how many files have been uploaded (e.g., "3 of 5 completed")
5. **Given** some files in a batch fail to upload, **When** the batch completes, **Then** the system shows a summary with successful uploads and errors for failed ones, allowing retry of failures

---

### Edge Cases

- What happens when a user tries to upload a file larger than Jira's attachment size limit (typically 10-100MB)?
  - System validates file size before upload and shows error message with the limit
  - For large markdown files, only the text content is sent in the description field (not as attachment)

- What happens when the Jira API is temporarily unavailable or rate-limited?
  - System shows a retry-able error message with exponential backoff
  - Failed uploads can be retried from the UI without reconfiguring
  - System logs errors for administrative monitoring

- What happens when a user modifies a file locally while it's being uploaded to Jira?
  - Upload uses the file content at the time of initiation
  - Users are notified if the file has been modified after upload completes

- What happens when a Jira issue linked to an artifact is moved to a different project or deleted?
  - System detects 404/403 errors and marks the link as broken
  - User can re-link to a different issue or remove the link
  - Broken links are visually distinguished from valid links

- What happens when two users upload the same artifact to Jira simultaneously?
  - If creating new issues, two separate issues are created (expected behavior)
  - If updating an existing issue, the last write wins (Jira's default behavior)
  - System displays the Jira issue key immediately after upload to prevent confusion

- What happens when a user doesn't have permission to create issues in the configured Jira project?
  - Jira API returns a 403 error with specific permission details
  - System displays a user-friendly error: "You don't have permission to create issues in project PROJ. Contact your Jira administrator."

- What happens when converting complex markdown (tables, code blocks, images) to Jira format?
  - System uses a markdown-to-Jira converter supporting common elements
  - Unsupported elements are preserved as code blocks or plain text
  - Images can be uploaded as attachments (future enhancement)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST allow users to upload individual artifact files from Shared Artifacts directory to Jira as new issues
- **FR-002**: System MUST allow users to update existing Jira issues with modified artifact content
- **FR-003**: System MUST display visual indicators (badges) showing which artifacts have been synced to Jira
- **FR-004**: System MUST display the Jira issue key (e.g., "PROJ-123") on synced artifact files
- **FR-005**: System MUST allow users to open Jira issues directly from the vTeam UI
- **FR-006**: System MUST persist Jira link metadata (artifact path to issue key mapping) across session restarts
- **FR-007**: System MUST convert markdown content to Jira-compatible format (wiki markup or Atlassian Document Format)
- **FR-008**: System MUST securely store Jira credentials (URL, project key, email, API token) in Kubernetes secrets
- **FR-009**: System MUST provide a connection test function to validate Jira credentials
- **FR-010**: System MUST handle authentication errors and provide clear error messages to users
- **FR-011**: System MUST support both Jira Cloud and Jira Server/Data Center REST API versions [NEEDS CLARIFICATION: which specific API versions - v2, v3, or both?]
- **FR-012**: System MUST validate file size before upload and reject files exceeding [NEEDS CLARIFICATION: what is the maximum file size limit?]
- **FR-013**: System MUST provide a context menu on artifact files with Jira-related actions (Upload, View in Jira, Copy Link)
- **FR-014**: System MUST display tooltips on Jira badges showing issue summary and status
- **FR-015**: System MUST support selecting multiple artifacts for bulk upload operations
- **FR-016**: System MUST handle network failures gracefully with retry mechanisms
- **FR-017**: System MUST log all Jira integration operations for audit and debugging purposes
- **FR-018**: System MUST allow users to unlink an artifact from a Jira issue
- **FR-019**: System MUST detect when linked Jira issues are deleted or inaccessible and mark links as broken
- **FR-020**: System MUST scope Jira credentials to individual projects (not shared across all projects)

### Key Entities

- **JiraLink**: Represents the association between an artifact file and a Jira issue
  - Attributes: artifactPath (string), jiraIssueKey (string), jiraIssueUrl (string), syncedAt (timestamp), status (active/broken)
  - Stored in session metadata under `jiraLinks` array field
  - Persisted in Kubernetes AgenticSession custom resource

- **JiraCredentials**: Represents the Jira connection configuration for a project
  - Attributes: jiraUrl (string), projectKey (string), userEmail (string), apiToken (secret string)
  - Stored in Kubernetes secrets with naming pattern: `{project-name}-jira-credentials`
  - Encrypted at rest by Kubernetes secrets management
  - Retrieved by backend API when making Jira API calls

- **JiraIssue**: Represents a Jira issue (external entity, referenced by integration)
  - Key attributes: issueKey (string), summary (string), description (string), status (string), url (string)
  - Not stored in vTeam database; retrieved via Jira REST API when needed
  - Used for displaying tooltips and validating links

- **ArtifactFile**: Existing entity extended with Jira sync metadata
  - New optional attribute: jiraLink (JiraLink reference)
  - Used for rendering Jira badges in the UI
  - Path serves as the unique identifier for linking to Jira issues

- **UploadOperation**: Represents a Jira upload operation (transient entity)
  - Attributes: artifactPath, operationType (create/update), jiraIssueKey (for updates), status (pending/success/failed), errorMessage
  - Used for tracking batch uploads and displaying progress
  - Not persisted beyond the operation lifecycle

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can successfully upload an artifact to Jira and create a new issue in under 10 seconds (95th percentile)
- **SC-002**: Jira sync status badges are visible on artifact files within 2 seconds of upload completion
- **SC-003**: Users can open a Jira issue from the UI with a single click, and the correct issue loads in a new tab 100% of the time
- **SC-004**: Jira link metadata persists across session restarts with 100% accuracy (no data loss)
- **SC-005**: System handles network failures gracefully, with successful retry within 30 seconds for 90% of transient failures
- **SC-006**: Markdown to Jira format conversion preserves readability for 95% of common markdown elements (headers, lists, code blocks, links, bold/italic)
- **SC-007**: Users can configure and validate Jira credentials in under 2 minutes, with clear error messages for common configuration mistakes
- **SC-008**: Bulk upload of 10 artifacts completes in under 60 seconds with a visible progress indicator
- **SC-009**: 80% of users who configure Jira integration successfully upload at least one artifact within their first session
- **SC-010**: Zero security incidents related to credential exposure or unauthorized Jira access within 90 days of release
- **SC-011**: System provides actionable error messages for 100% of common failure scenarios (invalid credentials, permission errors, network timeouts, missing configuration)
- **SC-012**: Support tickets related to Jira integration issues are resolved using logged audit information 90% of the time without requiring code changes
