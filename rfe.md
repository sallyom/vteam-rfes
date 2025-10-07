# Jira Integration for Session Artifacts Using Atlassian MCP Server

**Feature Overview:**

*Enable automatic publishing of session artifacts (spec.md, tasks.md, plan.md) to Jira issues using the Atlassian MCP Server. After each workflow session completes, the resulting artifacts are automatically pushed to a designated Jira issue, creating a seamless bridge between AI-assisted planning workflows and tracked work items in Jira.*

This feature integrates the official Atlassian MCP (Model Context Protocol) Server to create a robust, secure, and maintainable connection between vTeam workflow sessions and Jira issue tracking. Users benefit from automatic synchronization of planning artifacts without manual copy-paste or file uploads, ensuring Jira issues remain up-to-date with the latest specifications, tasks, and implementation plans.

**Goals:**

*Provide seamless integration between vTeam workflow sessions and Jira, ensuring that planning artifacts are automatically available in Jira issues for tracking and collaboration.*

* **Automated Artifact Publishing**: At the end of each session, automatically push spec.md, tasks.md, and plan.md to a linked Jira issue
* **Enhanced Security**: Use OAuth 2.0 authentication with TLS 1.2+ encryption via Atlassian MCP Server
* **Reduced Manual Work**: Eliminate manual copy-paste of session artifacts into Jira
* **Better Traceability**: Maintain clear linkage between workflow sessions and Jira issues
* **Future Extensibility**: Build on MCP standard for potential expansion to other Atlassian products

**Target Users**: Development teams, product managers, and architects who use vTeam for AI-assisted workflow planning and Jira for issue tracking. Teams benefit from having specifications and tasks automatically synchronized to their Jira workflow.

**Expected Outcome**: When a workflow session completes, the generated artifacts (spec.md, tasks.md, plan.md) are automatically pushed to the associated Jira issue as attachments or formatted in the issue description/comments, creating a single source of truth.

**Out of Scope:**

*Items explicitly excluded from this Feature:*

* Bi-directional sync (Jira updates do not flow back to vTeam sessions)
* Real-time synchronization during session execution
* Support for on-premise Jira instances (Atlassian MCP Server requires Atlassian Cloud)
* Publishing arbitrary workspace files (only spec.md, tasks.md, plan.md)
* Custom Jira field mapping beyond standard fields
* Jira webhook handling for status updates
* Confluence integration (future RFE)
* Automatic issue creation (assumes issue already exists or is created separately)

**Requirements:**

*Specific capabilities that must be delivered:*

* **[MVP] MCP Client Integration**: Integrate Atlassian MCP Server client library into vTeam backend
* **[MVP] OAuth Authentication**: Implement OAuth 2.0 authentication flow for Atlassian Cloud
* **[MVP] Artifact Detection**: Detect when session artifacts (spec.md, tasks.md, plan.md) are generated
* **[MVP] Jira Issue Linking**: Allow users to link workflow sessions to Jira issue keys
* **[MVP] Artifact Publishing**: Push session artifacts to linked Jira issue via MCP Server
* **[MVP] Error Handling**: Provide clear error messages when publishing fails
* **Session Completion Trigger**: Trigger artifact publishing when session reaches completion state
* **Configuration UI**: Provide UI for configuring Atlassian OAuth credentials and default Jira project
* **Artifact Formatting**: Format markdown artifacts appropriately for Jira (maintain readability)
* **Publishing Strategy**: Support either attachments, issue description updates, or comments (configurable)
* **Rate Limit Handling**: Implement graceful handling of Atlassian MCP rate limits
* **Audit Logging**: Log all Jira publishing operations for troubleshooting and compliance

**Done - Acceptance Criteria:**

*The feature is complete when:*

* Users can configure Atlassian Cloud OAuth credentials in vTeam settings
* Users can link a workflow session to a Jira issue key
* When a session completes, spec.md, tasks.md, and plan.md are automatically published to the linked Jira issue
* Published artifacts are readable and properly formatted in Jira
* Users receive confirmation notification when artifacts are successfully published
* Clear error messages are shown if publishing fails (authentication, permissions, connectivity)
* Session metadata tracks Jira issue linkage and publishing status
* Administrators can monitor Jira integration health and publishing operations
* Documentation covers setup, usage, and troubleshooting
* Integration tests validate end-to-end artifact publishing workflow

**Use Cases - i.e. User Experience & Workflow:**

**Main Success Scenario: Publishing Session Artifacts to Jira**

1. Product Manager creates a Jira issue "PROJ-456: Implement Dark Mode Support"
2. Product Manager starts a new vTeam workflow session for the feature
3. During session setup, Product Manager enters Jira issue key "PROJ-456"
4. AI agents generate spec.md, tasks.md, and plan.md during the session
5. Product Manager reviews and approves the artifacts
6. Session is marked as complete
7. vTeam backend triggers Jira publishing workflow
8. Backend connects to Atlassian MCP Server using configured OAuth credentials
9. Backend pushes spec.md, tasks.md, and plan.md to PROJ-456 (as attachments or formatted comments)
10. Jira issue PROJ-456 is updated with the three artifacts
11. Product Manager receives notification: "Session artifacts published to PROJ-456"
12. Product Manager opens PROJ-456 in Jira and sees all artifacts attached

**Alternative Flow 1: Jira Integration Not Configured**

1-3. Same as main scenario
4. Session proceeds without Jira linkage
5. Session completes
6. System detects no Jira configuration
7. Artifacts remain in vTeam only
8. Optional: User receives notification suggesting Jira integration setup

**Alternative Flow 2: OAuth Token Expired**

1-7. Same as main scenario
8. Backend attempts to connect with expired OAuth token
9. Atlassian returns 401 Unauthorized
10. System logs authentication failure
11. User receives error: "Failed to publish to Jira: Authentication expired. Please re-authenticate in Settings."
12. User re-authenticates via Settings
13. User manually triggers re-publish from session history
14. Artifacts are successfully published

**Alternative Flow 3: Jira Issue Not Found**

1-7. Same as main scenario
8. Backend connects to Atlassian MCP Server
9. Backend attempts to access PROJ-456
10. Jira returns 404 (issue doesn't exist or user lacks permission)
11. System logs error with context
12. User receives error: "Failed to publish to PROJ-456: Issue not found or insufficient permissions."
13. User verifies issue key and permissions
14. User updates session with correct issue key and triggers re-publish

**Alternative Flow 4: Network/MCP Server Failure**

1-7. Same as main scenario
8. Backend attempts to connect to Atlassian MCP Server
9. Connection times out or MCP Server is unavailable
10. System retries 3 times with exponential backoff
11. All retries fail
12. Session is marked with "pending Jira publish" status
13. User receives notification: "Session complete. Jira publishing failed and will be retried automatically."
14. Background job retries publishing every 30 minutes
15. When MCP Server recovers, artifacts are successfully published

**Documentation Considerations:**

*Documentation updates required:*

* **User Guide**: How to link workflow sessions to Jira issues and configure Atlassian OAuth
* **Setup Guide**: Step-by-step OAuth app creation in Atlassian Cloud with screenshots
* **Admin Guide**: Deploying and configuring Atlassian MCP Server connection, environment variables, and secrets
* **Prerequisites**: Atlassian Cloud requirement, supported Jira issue types, permission requirements
* **Rate Limits**: Atlassian MCP Server rate limits and impact on high-volume usage
* **Troubleshooting**: Common errors (authentication, permissions, connectivity) and resolution steps
* **Architecture Docs**: System architecture diagrams showing session completion triggers and MCP integration
* **API Reference**: Backend API endpoints for Jira configuration and manual artifact publishing

**Questions to answer:**

*Refinement and architectural questions:*

1. **Session Completion Detection**: How do we detect when a session is "complete"? User action? Automatic detection after artifact generation? Timeout-based?
2. **Publishing Format**: Should artifacts be uploaded as attachments, added as comments, or merged into issue description? What's most useful for users?
3. **Issue Linking Timing**: When should users link Jira issues? During session creation? After completion? Both?
4. **Multiple Sessions to One Issue**: Should we support multiple vTeam sessions publishing to the same Jira issue? How do we handle versioning/overwrites?
5. **Artifact Versioning**: If a session is re-run or artifacts are regenerated, should we overwrite or create new versions in Jira?
6. **MCP Client Language**: Use Python MCP client via subprocess/service or build native Go client? What's the performance/maintenance trade-off?
7. **OAuth Token Storage**: Where should refresh tokens be stored? Per-project secrets? Global configuration? User-specific credentials?
8. **Retry Strategy**: How aggressively should we retry failed publishing? Background jobs? Manual triggers?
9. **Partial Success**: What if only some artifacts publish successfully? Mark as partial success or full failure?
10. **Issue Type Restrictions**: Should we restrict publishing to certain Jira issue types (Story, Task)? Or allow any type?
11. **Bulk Operations**: Should we support publishing artifacts from multiple sessions to multiple issues in one operation?
12. **Markdown to Jira Formatting**: How do we handle markdown-to-Jira format conversion? Use Atlassian Document Format (ADF)?

**Background & Strategic Fit:**

*Additional context:*

vTeam provides AI-assisted workflows for creating software feature specifications, breaking them into tasks, and generating implementation plans. The typical workflow generates three key artifacts:

1. **spec.md**: Feature specification with requirements, acceptance criteria, and use cases
2. **tasks.md**: Breakdown of implementation tasks with dependencies and estimates
3. **plan.md**: Technical implementation plan with architecture decisions and approach

These artifacts form the foundation for development work but must be tracked in issue management systems like Jira for project visibility, collaboration, and sprint planning. Currently, users must manually copy content from vTeam sessions into Jira issues, which is time-consuming and error-prone.

Atlassian's MCP Server, announced in early 2025, provides an official, supported integration path for AI tools to interact with Jira programmatically. Key benefits:

* **OAuth 2.0**: More secure than API tokens, with granular permission scopes
* **Rate Limiting**: Built-in protections prevent overwhelming Jira instances
* **Maintenance**: Atlassian maintains the integration, reducing vTeam's maintenance burden
* **Future Growth**: MCP standard enables potential expansion to Confluence, Bitbucket, and other tools

By automating artifact publishing to Jira, vTeam reduces friction in the planning-to-execution transition and ensures development teams have immediate access to AI-generated planning artifacts in their existing workflow tools.

The Model Context Protocol (MCP) is an open standard developed by Anthropic for enabling AI applications to securely access external data sources and tools. Adopting MCP positions vTeam to integrate with a growing ecosystem of MCP-enabled services.

**Customer Considerations**

*Customer-specific considerations:*

* **Atlassian Cloud Requirement**: Only Atlassian Cloud is supported (not Jira Server/Data Center). Customers on on-premise deployments cannot use this feature. This must be clearly documented and communicated during sales/onboarding.

* **OAuth Setup Complexity**: OAuth requires creating Atlassian OAuth apps and managing credentials. Small teams or less technical users may need additional support. Provide comprehensive documentation with screenshots and video walkthrough.

* **Rate Limits**: Atlassian Standard plans have lower rate limits than Premium/Enterprise. High-volume users or large enterprises running many sessions may hit limits. Consider implementing client-side rate limiting and usage dashboards.

* **Data Residency**: Requests flow through Atlassian's MCP Server infrastructure. Customers with strict data residency requirements should understand data path. All data is encrypted in transit (TLS 1.2+).

* **Red Hat Internal**: For Red Hat's internal issues.redhat.com (Jira), verify OAuth app registration process and test MCP Server compatibility with Red Hat's Atlassian configuration.

* **Permission Management**: Users must have appropriate Jira permissions to create/update issues. Organizations with strict permission models may need to adjust policies or create service accounts.

* **Artifact Size**: Jira has size limits for attachments and comment lengths. Very large artifacts (> 10MB) may fail. Implement size validation and provide clear error messages.

* **Migration Path**: If vTeam previously had custom Jira integration, provide migration documentation and tooling to help users reconfigure with OAuth.

* **Cost Impact**: Verify Atlassian licensing impact. Some API operations may count against Atlassian usage limits or licensing tiers.

* **Compliance**: For regulated industries (finance, healthcare), ensure Jira integration meets compliance requirements (audit logging, data encryption, access controls).
