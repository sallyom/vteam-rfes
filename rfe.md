# Jira Integration for vTeam Feature Specification Management

**Feature Overview:**
*Enable vTeam users to seamlessly push and synchronize feature specifications to Jira, automatically creating Feature Requests with spec.md content and attaching all generated specification artifacts from the spec directory. This integration bridges the gap between vTeam's AI-powered specification generation workflow and Jira's enterprise project management capabilities, ensuring design artifacts are automatically tracked in the project management system of record.*

**Goals:**

*Provide seamless integration between vTeam's specification workflow and Jira project management:*

* **Automated Spec Sync**: Users can push completed spec.md files to Jira as Feature Requests with a single command, eliminating manual copy-paste workflows
* **Complete Artifact Tracking**: All specification artifacts (spec.md, tasks.md, plan.md, architecture diagrams, etc.) are automatically attached to the Jira Feature Request for full context
* **Bidirectional Status Updates**: Changes to Jira Feature Request status are reflected back in vTeam, maintaining sync between specification and project management
* **Team Collaboration**: Development teams working in Jira gain immediate access to AI-generated specifications and design artifacts
* **Traceability**: Establishes clear linkage between vTeam specification sessions and Jira Feature Requests for audit and tracking purposes

**Out of Scope:**

*The following items are explicitly excluded from this feature:*

* Bidirectional content synchronization (edits in Jira are not synced back to vTeam spec files)
* Integration with other project management tools (e.g., Azure DevOps, Linear, GitHub Projects)
* Automated creation of Jira sub-tasks, epics, or stories from vTeam tasks.md
* Real-time collaboration features or live editing
* Jira workflow automation or custom field mapping beyond basic Feature Request creation
* Migration or bulk import of existing specifications
* Integration with Jira Service Management or Jira Align

**Requirements:**

*Core capabilities required for MVP and full feature delivery:*

* **MVP-001** *(MVP)*: System MUST provide a command or UI action to push spec.md to Jira as a new Feature Request
* **MVP-002** *(MVP)*: System MUST authenticate with Jira using API tokens or OAuth 2.0
* **MVP-003** *(MVP)*: System MUST create a Jira Feature Request (or Issue of type "Feature") with spec.md content as the description
* **MVP-004** *(MVP)*: System MUST attach all files from the specification directory (e.g., .specify/specs/{feature-name}/) as Jira attachments
* **MVP-005** *(MVP)*: System MUST store the Jira issue key (e.g., PROJ-123) in vTeam session metadata for future reference
* **REQ-006**: System SHOULD allow users to configure target Jira project, issue type, and custom field mappings
* **REQ-007**: System SHOULD support updating existing Jira Feature Requests when spec.md is modified
* **REQ-008**: System SHOULD retrieve and display Jira Feature Request status within vTeam UI/CLI
* **REQ-009**: System SHOULD validate Jira connection and credentials before attempting to sync
* **REQ-010**: System SHOULD provide clear error messages when sync fails (authentication, network, permissions)
* **REQ-011**: System SHOULD support markdown formatting conversion from spec.md to Jira wiki markup or maintain markdown if Jira supports it
* **REQ-012**: System SHOULD allow users to specify Jira components, labels, and assignee during sync
* **REQ-013**: System MUST log all Jira sync operations for audit purposes
* **REQ-014**: System SHOULD handle large specification artifacts (>10MB total) by compressing or selective attachment

**Done - Acceptance Criteria:**

*Feature is considered complete when the following criteria are met:*

* A vTeam user can execute a command (e.g., `vteam jira push` or click "Push to Jira" in UI) to sync a completed spec.md to Jira
* The command successfully authenticates with Jira using configured credentials (API token or OAuth)
* A new Jira Feature Request is created in the configured project with:
  * Summary derived from spec.md title
  * Description populated with spec.md content (markdown preserved or converted to Jira format)
  * All files from the spec directory attached (spec.md, tasks.md, plan.md, diagrams, etc.)
  * Appropriate metadata (components, labels) set based on user configuration
* The Jira issue key is stored in vTeam and displayed to the user
* User can view the Jira issue link directly from vTeam
* Subsequent updates to spec.md can be pushed to update the same Jira Feature Request
* Error scenarios (authentication failure, network timeout, invalid project) display clear error messages
* Documentation includes setup instructions for Jira API token generation and configuration
* Integration works with Jira Cloud and Jira Server/Data Center

**Use Cases - i.e. User Experience & Workflow:**

### Use Case 1: Initial Specification Push to Jira

**Actor**: Product Owner / Engineer using vTeam

**Main Success Scenario**:
1. User completes feature specification using vTeam (runs /specify, /plan, /tasks commands)
2. Specification artifacts are generated in `.specify/specs/feature-name/` directory
3. User reviews spec.md and confirms it's ready for project management tracking
4. User executes `vteam jira push --project PROJ --spec feature-name` or clicks "Push to Jira" in UI
5. System authenticates with Jira using stored API token
6. System creates new Feature Request in PROJ with spec.md as description
7. System attaches all files from `.specify/specs/feature-name/` (spec.md, tasks.md, diagrams, etc.)
8. System displays success message: "Created PROJ-456: Feature Name - https://jira.company.com/browse/PROJ-456"
9. User clicks link to view Feature Request in Jira
10. Development team in Jira can now review specification and begin planning

**Alternative Flow 1** (Authentication Failure):
- At step 5, if API token is invalid or expired
- System displays: "Jira authentication failed. Please update API token in Settings"
- User navigates to Settings > Integrations > Jira and updates credentials

**Alternative Flow 2** (Update Existing Feature):
- At step 4, if spec was previously pushed (Jira key exists in metadata)
- System prompts: "PROJ-456 already exists. Update existing Feature Request? [y/N]"
- User confirms, system updates description and re-uploads attachments

### Use Case 2: Status Synchronization

**Actor**: Engineer monitoring feature status

**Main Success Scenario**:
1. User opens vTeam session for previously synced specification
2. System displays: "Linked to PROJ-456 [Status: In Progress]"
3. User executes `vteam jira status feature-name`
4. System fetches current status, assignee, and comments from Jira
5. System displays updated information in vTeam UI/CLI

### Use Case 3: Configuration Setup

**Actor**: Team Lead setting up Jira integration

**Main Success Scenario**:
1. User navigates to vTeam Settings > Integrations > Jira
2. User enters Jira URL (e.g., https://company.atlassian.net)
3. User enters Jira API Token (generated from Atlassian Account Settings)
4. User enters email associated with API token
5. User configures default project key (e.g., PROJ)
6. User selects default issue type (Feature, Story, Task)
7. User optionally configures component and label mappings
8. User clicks "Test Connection"
9. System validates credentials and displays available projects
10. User saves configuration

**Documentation Considerations:**

*Documentation required for successful feature adoption:*

* **Setup Guide**: Step-by-step instructions for generating Jira API tokens (Cloud and Server/Data Center)
* **Configuration Reference**: Complete reference for all Jira integration settings (URL, credentials, project mappings, field mappings)
* **Command Reference**: Documentation for CLI commands (`vteam jira push`, `vteam jira status`, `vteam jira config`)
* **UI Guide**: Screenshots and walkthrough for UI-based Jira sync workflow
* **Troubleshooting Guide**: Common error scenarios and resolutions (authentication, permissions, network)
* **Security Best Practices**: Guidance on API token storage, rotation, and access control
* **Markdown Conversion Reference**: How vTeam markdown is converted to Jira wiki markup or preserved
* **Link to Existing Docs**: This extends vTeam specification workflow documented in `.specify/` directory

**Questions to answer:**

*Refinement questions that need answers before implementation:*

* **Q1**: Should we use Atlassian MCP Server for Jira integration, or implement direct REST API calls?
  * *Consideration*: MCP Server provides standardized interface but adds dependency
* **Q2**: How should we handle Jira issue types that don't have a "Feature Request" type?
  * *Options*: Allow configurable issue type mapping, default to "Story" or "Task"
* **Q3**: Should spec updates in vTeam automatically sync to Jira, or require explicit user action?
  * *Trade-off*: Auto-sync provides convenience but may create unwanted noise in Jira
* **Q4**: How do we handle conflicting updates (spec changed in vTeam, description edited in Jira)?
  * *Options*: vTeam is source of truth (overwrite Jira), conflict detection, append mode
* **Q5**: Should we support attaching only specific files from spec directory, or always attach all?
  * *Consideration*: Large artifact sets may exceed Jira attachment limits
* **Q6**: How should we store Jira credentials - per-user, per-project, or global vTeam instance?
  * *Security*: Per-user is most secure, global is most convenient
* **Q7**: Should we create Jira comments for significant spec changes (append-only audit trail)?
* **Q8**: Do we need to support Jira custom fields specific to Feature Requests?
  * *Example*: Target Release, Story Points, Business Value fields
* **Q9**: Should the integration support multiple Jira instances (e.g., internal and customer Jira)?
* **Q10**: How should we handle specification artifacts that are generated after initial Jira push?
  * *Options*: Manual re-sync, automatic detection and update

**Background & Strategic Fit:**

*Context and strategic justification for this feature:*

vTeam is designed as a Kubernetes-native AI automation platform that combines Claude Code CLI with multi-agent collaboration to generate high-quality feature specifications, implementation plans, and task breakdowns. The platform currently stores these artifacts in the `.specify/` directory structure as markdown files and diagrams.

Many enterprise development teams use Jira as their central project management and issue tracking system. While vTeam excels at AI-powered specification generation, development teams need these specifications tracked and managed within their existing Jira workflows. Currently, users must manually copy spec content to Jira and attach files, creating friction and reducing adoption.

This integration aligns with vTeam's goal of becoming the "Ambient Agentic Runner" for development teams by:
* **Reducing context switching**: Developers can work in vTeam for specification, Jira for project management
* **Improving traceability**: Clear linkage between AI-generated specs and Jira Feature Requests
* **Accelerating feature planning**: Specifications move immediately from vTeam to team backlogs
* **Enabling enterprise adoption**: Jira integration is table-stakes for enterprise development teams

The feature builds on vTeam's existing specification workflow (slash commands: /specify, /plan, /tasks) and extends it with project management integration. This complements vTeam's core strength (AI-powered specification generation) rather than attempting to replace Jira's project management capabilities.

**Customer Considerations**

*Customer-specific factors that influence feature design:*

* **Enterprise Security Requirements**: Many enterprises restrict API access and require OAuth 2.0 rather than API tokens. Implementation must support both authentication methods.
* **Jira Cloud vs Server/Data Center**: API differences between Jira Cloud (REST API v3) and Server/Data Center (REST API v2) require abstraction layer or separate implementations.
* **Custom Jira Workflows**: Customers may have custom issue types, workflows, and required fields beyond standard Jira. Configuration must be flexible.
* **Attachment Size Limits**: Jira Cloud has 10MB per-file and 100MB total attachment limits. Implementation must handle or warn about large specification artifacts.
* **Multi-Project Scenarios**: Large enterprises may have multiple Jira projects per team. Configuration must support project-per-feature or user selection.
* **Compliance & Audit**: Regulated industries require audit trails for specification changes. Integration must log all sync operations with timestamps and user identity.
* **Air-Gapped Environments**: Some customers run Jira Server in air-gapped environments. Integration must gracefully handle network-isolated scenarios.
* **SSO Integration**: Customers using Atlassian Access with SSO may require OAuth flows compatible with their identity provider.
* **Multi-Tenancy**: vTeam supports project-scoped namespaces. Jira configuration may need to be project-scoped or user-scoped, not global.
* **Developer Experience**: Engineers using vTeam may not have Jira project admin permissions. Integration must work with standard user permissions (create issues, add attachments).
