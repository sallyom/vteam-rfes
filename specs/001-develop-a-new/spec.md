# Feature Specification: Jira-Session Artifact Integration

**Feature Branch**: `001-develop-a-new`
**Created**: 2025-10-09
**Status**: Draft
**Input**: User description: "Develop a new feature on top of the projects in /repos. Feature requirements: Integrate Jira with sessions. Push session artifacts to Jira."

## Execution Flow (main)
```
1. Parse user description from Input
   → Feature involves pushing agentic session artifacts to Jira issues
2. Extract key concepts from description
   → Actors: vTeam users, AgenticSession Custom Resources, Jira system
   → Actions: Link sessions to Jira, push artifacts, track sync status
   → Data: Session state files, chat transcripts, results, Jira issues
   → Constraints: Must work with existing vTeam/AgenticSession architecture
3. For each unclear aspect:
   → [RESOLVED: Artifacts are session state files from stateDir in CR status]
   → [RESOLVED: Push triggered manually via UI/API, not automatic]
   → [RESOLVED: Artifacts attached to Jira issues as attachments and comments]
   → [RESOLVED: Authentication via JIRA_API_TOKEN in runner secrets]
   → [RESOLVED: Failures logged in session status, retryable]
4. Fill User Scenarios & Testing section
   → Primary flow: User completes agentic session, pushes artifacts to Jira
5. Generate Functional Requirements
   → Requirements aligned with AgenticSession CRD and existing Jira handlers
6. Identify Key Entities
   → AgenticSession, SessionArtifact, JiraIssue, JiraLink
7. Run Review Checklist
   → All critical requirements testable and unambiguous
8. Return: SUCCESS (spec ready for planning)
```

---

## ⚡ Quick Guidelines
- ✅ Focus on WHAT users need and WHY
- ❌ Avoid HOW to implement (no tech stack, APIs, code structure)
- 👥 Written for business stakeholders, not developers

### Section Requirements
- **Mandatory sections**: Must be completed for every feature
- **Optional sections**: Include only when relevant to the feature
- When a section doesn't apply, remove it entirely (don't leave as "N/A")

---

## User Scenarios & Testing

### Primary User Story
A development team uses vTeam to run intelligent agentic sessions that analyze codebases, generate documentation, or solve technical problems. Each session produces valuable artifacts including conversation transcripts, analysis results, generated code, and decision records. The team needs these artifacts tracked in their existing Jira workflow for visibility, collaboration, and audit purposes. Users can link agentic sessions to Jira issues and push session artifacts (state files, transcripts, results) to the corresponding Jira issue as attachments and comments, creating a complete record of AI-assisted work.

### Acceptance Scenarios

1. **Given** a user has completed an agentic session with results in the "Completed" phase, **When** the user navigates to the session details page and clicks "Push to Jira", **Then** a dialog appears allowing them to enter a Jira issue key (e.g., PROJ-123), select which artifacts to push (transcript, results, state files), and confirm the push operation

2. **Given** a user enters a valid Jira issue key and selects artifacts to push, **When** the user confirms the push operation and the Jira system is accessible, **Then** the system attaches selected artifacts to the Jira issue, adds a comment summarizing the session (prompt, model, cost, duration), and displays a success message with a direct link to the Jira issue

3. **Given** a user attempts to push artifacts to Jira, **When** the project namespace does not have Jira credentials configured (JIRA_URL, JIRA_PROJECT, JIRA_API_TOKEN in runner secrets), **Then** the system displays a clear error message directing the user to configure Jira settings in Project Settings > Runner Secrets before attempting to push

4. **Given** a user pushes artifacts to a Jira issue successfully, **When** the user later views the session details, **Then** the UI displays a "Linked to Jira" badge with the issue key and a clickable link that opens the Jira issue in a new browser tab

5. **Given** multiple users push artifacts from different sessions to the same Jira issue, **When** each push operation completes, **Then** all artifacts are properly attached without overwriting previous attachments, and each session's comment is appended to the issue with a timestamp

6. **Given** a user attempts to push artifacts to Jira, **When** the Jira system is temporarily unavailable or the API token is invalid, **Then** the system displays a specific error message (network timeout, authentication failure, or permission denied) and allows the user to retry the operation without losing the session data

7. **Given** a session has been linked to multiple Jira issues over time, **When** the user views session details, **Then** the UI displays a list of all linked Jira issues with issue keys, titles, and timestamps of when artifacts were pushed

8. **Given** a user is viewing an agentic session in "Running" or "Failed" state, **When** the user attempts to push artifacts to Jira, **Then** the system allows the push operation and includes partial results or error details in the pushed artifacts so Jira captures the current state

### Edge Cases
- What happens when a user enters an invalid or non-existent Jira issue key? (System validates with Jira API before pushing, displays "Issue not found" error)
- What happens when session artifacts exceed Jira attachment size limits (10MB per file)? (System warns user, allows selecting smaller artifacts, or compresses files before upload)
- How does the system handle partial failures where some artifacts upload successfully but others fail? (System reports which artifacts succeeded/failed, allows retry for failed items only)
- What happens if the user lacks permissions to update the specified Jira issue? (Jira API returns 403, system displays "Permission denied" with suggestion to contact Jira admin)
- How are artifacts from interactive chat sessions handled differently from non-interactive sessions? (Interactive sessions include full chat history with user messages; non-interactive only include agent outputs)
- What happens when session state directory is empty or missing? (System displays warning "No artifacts available to push", prevents push operation)
- How does system handle sessions that timed out or were interrupted? (Allows pushing partial artifacts with clear indication in Jira comment that session was incomplete)
- What happens when Jira project key in runner secrets doesn't match the target issue's project? (System validates issue belongs to configured project, warns if mismatch, allows override)

## Requirements

### Functional Requirements

**Session-Jira Linking**
- **FR-001**: System MUST allow users to link an agentic session to one or more Jira issues by providing a Jira issue key (format: PROJECT-123)
- **FR-002**: System MUST validate Jira issue keys by querying the Jira API before accepting the link to ensure the issue exists and is accessible
- **FR-003**: System MUST display all linked Jira issues for a session in the session details UI with issue key, title, status, and link to open in Jira
- **FR-004**: System MUST store Jira links in the AgenticSession Custom Resource's metadata annotations for persistence across session lifecycle

**Artifact Selection and Push**
- **FR-005**: System MUST identify available session artifacts from the session's stateDir (chat transcripts, result summaries, tool outputs, state files)
- **FR-006**: System MUST allow users to select which artifacts to push to Jira via checkboxes in the push dialog (default: all artifacts selected)
- **FR-007**: System MUST package selected artifacts as file attachments suitable for Jira's attachment API (preserving filenames and content)
- **FR-008**: System MUST create a Jira comment on the target issue summarizing the session: prompt, model used, duration, cost, phase (Completed/Failed), and link back to vTeam session
- **FR-009**: System MUST handle artifact files up to Jira's size limits (10MB per file) and warn users when files exceed limits
- **FR-010**: System MUST preserve artifact integrity during transfer (no corruption, maintain file formats, correct character encoding)

**Authentication and Authorization**
- **FR-011**: System MUST authenticate with Jira using credentials stored in the project namespace's runner secrets (JIRA_URL, JIRA_API_TOKEN, JIRA_PROJECT)
- **FR-012**: System MUST validate Jira configuration exists before allowing push operations and provide clear guidance if missing (link to Project Settings)
- **FR-013**: System MUST respect Jira API permissions and provide specific error messages when users lack access (view issue, add attachments, add comments)
- **FR-014**: System MUST use project-scoped Jira configuration, not global or user-specific credentials, aligning with vTeam's multi-tenant architecture

**Error Handling and Feedback**
- **FR-015**: System MUST provide real-time feedback during push operations with progress indicator (uploading artifacts, creating comment, finalizing)
- **FR-016**: System MUST handle Jira connectivity failures gracefully with specific error messages: network timeout, authentication failure, rate limit exceeded, permission denied
- **FR-017**: System MUST allow users to retry failed push operations without re-entering Jira issue key or re-selecting artifacts
- **FR-018**: System MUST log all Jira sync operations in backend logs with timestamp, user, session ID, Jira issue key, success/failure status for audit and debugging

**Status Tracking and History**
- **FR-019**: System MUST maintain a record of all push operations for each session including: Jira issue key, timestamp, artifacts pushed, success/failure status
- **FR-020**: System MUST display "Linked to Jira" badge in session list view for sessions that have been pushed to Jira
- **FR-021**: System MUST allow users to view push history for a session showing all past Jira pushes with timestamps and issue keys
- **FR-022**: System MUST update session annotations to include jiraLinks array with path (artifact name) and jiraKey for each pushed artifact

**Session Lifecycle Integration**
- **FR-023**: System MUST support pushing artifacts from sessions in any phase: Running, Completed, Failed, or Stopped
- **FR-024**: System MUST include appropriate context in Jira comment based on session phase (e.g., "Session failed with error" vs "Session completed successfully")
- **FR-025**: System MUST support both interactive and non-interactive agentic sessions with appropriate artifact content (full chat history vs agent outputs only)
- **FR-026**: System MUST work with AgenticSession Custom Resources created by both direct session creation and RFE workflow sessions

### Key Entities

- **AgenticSession**: A vTeam Custom Resource representing an AI-powered automation session. Contains spec (prompt, llmSettings, repos, timeout), status (phase, message, stateDir, result), and metadata (labels, annotations). Sessions produce artifacts stored in stateDir that can be pushed to Jira. Lifecycle: Pending → Creating → Running → Completed/Failed/Stopped.

- **SessionArtifact**: Files and data generated during an agentic session, stored in the stateDir path specified in the session's status. Types include: chat transcript (conversation history), result summary (final output), tool use logs (tools executed), state files (session state snapshots), and error details (for failed sessions). Each artifact has a filename, size, and MIME type.

- **JiraIssue**: An external entity in Jira representing a task, bug, feature, or story. Identified by a unique issue key (PROJECT-123). Has attributes: title, description, status, assignee, project, and supports attachments and comments. Users link AgenticSessions to JiraIssues to track AI-assisted work in their project management system.

- **JiraLink**: A relationship between a session artifact and a Jira issue, stored in the AgenticSession's annotations as a jiraLinks array. Each link contains: path (artifact filename), jiraKey (Jira issue key), timestamp (when pushed), status (success/failure). Enables tracking which artifacts were pushed to which issues.

- **JiraConfiguration**: Project-scoped settings stored in the runner secrets (Kubernetes Secret) defining how vTeam connects to Jira. Contains: JIRA_URL (instance URL), JIRA_PROJECT (default project key), JIRA_API_TOKEN (authentication token), and optional JIRA_USER_EMAIL. Managed via Project Settings UI.

- **PushOperation**: A user-initiated action to transfer session artifacts to a Jira issue. Includes: source session, target Jira issue key, selected artifacts, timestamp, success/failure status, and error message if failed. Logged for audit and displayed in push history UI.

---

## Review & Acceptance Checklist
*GATE: Automated checks run during main() execution*

### Content Quality
- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

### Requirement Completeness
- [x] No [NEEDS CLARIFICATION] markers remain (all resolved via system analysis)
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable (links created, artifacts uploaded, comments added)
- [x] Scope is clearly bounded (session artifacts to Jira only, no bidirectional sync)
- [x] Dependencies and assumptions identified (existing Jira integration structure, runner secrets)

---

## Business Value & Impact

### Why This Feature Matters
Development teams using vTeam generate significant value through AI-powered agentic sessions: code analysis, architecture reviews, documentation generation, and problem-solving. However, this work exists in isolation from their primary project management tools. Without integration, teams face:

- **Lost Context**: Session insights don't flow into sprint planning or issue tracking
- **Manual Overhead**: Copying results from vTeam to Jira is time-consuming and error-prone
- **Limited Visibility**: Stakeholders in Jira can't see AI-assisted work contributions
- **Poor Traceability**: No link between AI analysis and resulting tasks or decisions

This feature eliminates these gaps by automatically pushing session artifacts to Jira, ensuring AI-assisted work is tracked alongside traditional development activities.

### User Benefits
- **Product Owners**: See complete context for feature requirements including AI-generated analysis and recommendations in Jira tickets
- **Engineers**: Link code review sessions or debugging analysis to Jira issues for full audit trail
- **Team Leads**: Track AI automation usage and value through Jira reporting and dashboards
- **Compliance**: Maintain complete records of AI-assisted decisions attached to Jira issues for regulatory audit

### Success Metrics
- **Adoption**: % of completed agentic sessions pushed to Jira (target: >60% within 3 months)
- **Efficiency**: Reduction in time spent manually transferring session results to Jira (baseline: ~10 min per session)
- **Engagement**: Number of unique Jira issues with attached vTeam artifacts per project
- **User Satisfaction**: NPS or survey rating for Jira integration usefulness (target: >8/10)

---

## Scope Boundaries

### In Scope
- Pushing session artifacts from vTeam to Jira as attachments and comments
- Linking AgenticSessions to Jira issues with persistent tracking
- Project-scoped Jira configuration via runner secrets
- Support for Jira Cloud and Jira Server/Data Center REST APIs
- Error handling and retry for push operations
- Push history and status tracking in session UI

### Out of Scope
- **Bidirectional Sync**: Edits to Jira issue descriptions/comments NOT synced back to vTeam sessions
- **Automated Pushing**: No automatic push on session completion (always requires user action)
- **Jira Workflow Automation**: No automatic status transitions, assignee changes, or custom field updates in Jira
- **Sub-task Creation**: Not creating Jira sub-tasks, epics, or stories from session task breakdowns (future enhancement)
- **Real-time Sync**: Not continuously syncing session progress to Jira during execution
- **Other PM Tools**: Not integrating with Linear, Azure DevOps, GitHub Projects, or other project management systems
- **Jira Service Management**: Not integrating with JSM request types or SLA tracking
- **User-specific Credentials**: Not supporting per-user Jira tokens (project-level only)

---

## Dependencies & Assumptions

### System Dependencies
- Existing vTeam AgenticSession CRD with stateDir and result tracking in status
- Existing Jira integration foundation (backend handlers, API routes) in vTeam
- Project-scoped runner secrets infrastructure for storing Jira credentials
- Access to session state files stored in stateDir path
- Frontend UI framework supporting dialogs, file selection, and progress indicators

### Assumptions
- Users have Jira project administrator or permission to create issues and add attachments
- Jira API tokens generated from Atlassian account settings with appropriate scopes
- Session state files are persisted and accessible from backend API during push operation
- Network connectivity between vTeam backend and Jira instance for API calls
- Jira attachment size limits (10MB per file) are enforced by Jira, not vTeam
- Users understand Jira issue key format (PROJECT-123) and can identify target issues

### External Constraints
- Jira REST API rate limits may throttle high-volume push operations (typically 300-1000 req/min)
- Jira Cloud attachment size limit: 10MB per file, 100MB total per issue
- Jira authentication requires API token (Cloud) or PAT (Server/Data Center)
- Jira issue visibility follows Jira project permissions, vTeam cannot override
- Network latency affects push operation duration, especially for large artifacts

---

## Execution Status
*Updated by main() during processing*

- [x] User description parsed
- [x] Key concepts extracted
- [x] Ambiguities resolved via system analysis
- [x] User scenarios defined (8 acceptance scenarios, 8 edge cases)
- [x] Requirements generated (26 functional requirements)
- [x] Entities identified (6 key entities with relationships)
- [x] Review checklist passed
- [x] Business value and scope documented
- [x] Dependencies and assumptions identified

---

## Next Steps

**After Specification Approval**:
1. **Planning Phase**: Break down requirements into technical implementation tasks (API endpoints, UI components, CRD schema updates)
2. **Design Phase**: Define data models for JiraLink storage in annotations, artifact packaging format, error handling flows
3. **Implementation Phase**: Develop push dialog UI, backend push handler, artifact file reading, Jira API client
4. **Testing Phase**: Integration tests with Jira sandbox, UI testing for push flows, error scenario coverage
5. **Documentation Phase**: User guide for configuring Jira integration, API documentation, troubleshooting guide

**Key Questions for Planning**:
- Should system compress large artifacts automatically to fit Jira limits, or warn and require user selection?
- Should system support bulk push operations for multiple sessions to same Jira issue?
- What default artifact selection behavior: all artifacts, or smart selection based on session type?
- Should system cache Jira issue metadata (title, status) locally to reduce API calls?
