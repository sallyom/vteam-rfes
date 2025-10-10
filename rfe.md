# Update vTeam Documentation

**Feature Overview:**

Update the vTeam README and documentation to accurately reflect the current OpenShift-native architecture and deployment model. The existing documentation contains outdated information about LlamaDeploy, RFE builder demos, and Python/TypeScript local development that no longer matches the actual containerized, Kubernetes-native implementation.

**Goals:**

- Ensure all documentation accurately describes the current vTeam architecture
- Provide clear, straightforward instructions for deploying to OpenShift
- Remove obsolete references to demo applications and local Python workflows
- Update getting-started guides to match the actual deployment process
- Improve clarity and reduce cognitive load for new users

This benefits all vTeam users—developers, operators, and end-users—by providing accurate information that matches the actual system behavior. The difference between today and a world with this feature is that users can successfully deploy and use vTeam by following the documentation without encountering mismatches between documentation and reality.

**Out of Scope:**

- Creating new features or functionality
- Changing the architecture itself
- Adding new documentation sections beyond clarifying existing ones
- Translating documentation to other languages

**Requirements:**

1. **MVP**: Update main README.md to accurately reflect OpenShift-native architecture
2. **MVP**: Update docs/user-guide/getting-started.md with OpenShift deployment steps
3. **MVP**: Update docs/index.md with current architecture overview
4. Remove or update references to outdated demo applications
5. Ensure all code examples use correct commands (oc, make deploy, etc.)
6. Verify links between documentation files are correct
7. Maintain consistent terminology throughout documentation

**Done - Acceptance Criteria:**

From a user's perspective, the documentation update is complete when:

1. The main README accurately describes the OpenShift-native architecture
2. The getting-started guide successfully guides a user through deployment
3. No references to LlamaDeploy, RFE builder demos, or Python local dev remain (unless archived)
4. All commands and examples work as documented
5. The documentation structure is clear and easy to navigate
6. Links between documentation pages work correctly

**Use Cases - i.e. User Experience & Workflow:**

**Main Success Scenario:**

1. New user discovers vTeam repository
2. User reads README to understand what vTeam does
3. User follows getting-started guide to deploy on OpenShift
4. User successfully creates first project and agentic session
5. User references additional docs for advanced features

**Alternative Flow - Experienced OpenShift User:**

1. User reviews architecture section in README
2. User deploys using make deploy with custom .env configuration
3. User verifies deployment using provided oc commands
4. User accesses web interface and starts using vTeam

**Alternative Flow - Developer Contributing:**

1. Developer reads developer guide
2. Developer sets up local development with CRC
3. Developer makes changes and tests locally
4. Developer finds all documentation references accurate

**Documentation Considerations:**

This RFE IS documentation work. Key considerations:

- All documentation must be in Markdown format
- Follow existing documentation structure (docs/ folder with mkdocs.yml)
- Use consistent code block formatting with bash language hints
- Include clear section headings and navigation
- Provide troubleshooting sections with common issues
- Use admonitions (warnings, notes, tips) appropriately
- Keep language concise and action-oriented

**Questions to answer:**

1. ✅ What is the current deployment model? (OpenShift-native with pre-built images)
2. ✅ What components make up vTeam? (Frontend, Backend, Operator, Runner, Content Service)
3. ✅ What are the prerequisites for deployment? (OpenShift cluster, oc CLI, API keys)
4. ✅ How do users create and manage sessions? (Web UI with project-scoped namespaces)
5. ✅ What documentation should be archived vs updated? (Archive demo-specific docs, update core docs)

**Background & Strategic Fit:**

Accurate documentation is critical for:
- User adoption and success
- Reducing support burden
- Enabling community contributions
- Professional project impression
- Successful production deployments

This documentation update aligns with vTeam's strategic goal of being an enterprise-ready OpenShift-native AI automation platform. Accurate documentation builds trust and reduces friction for both internal Red Hat teams and external users.

**Customer Considerations**

- **Internal Red Hat Users**: Need accurate docs for RHOAI integration and internal deployments
- **OpenShift Customers**: Expect documentation that matches OpenShift best practices and terminology
- **Open Source Community**: Rely on documentation to evaluate and contribute to the project
- **Enterprise Operators**: Need clear deployment, security, and troubleshooting guidance

The documentation must serve both technical (developers, operators) and non-technical (product managers, architects) audiences with appropriate detail levels and clear navigation between beginner and advanced topics.

---

## Implementation Summary

The following files have been updated to address this RFE:

### Updated Files

1. **README.md** - Main repository README
   - Updated overview to describe OpenShift-native architecture
   - Corrected component descriptions (Next.js 15, Go backend, Claude Code SDK)
   - Updated prerequisites and deployment instructions
   - Added clear quick start with make deploy
   - Updated architecture flow diagram
   - Improved troubleshooting section
   - Added production considerations

2. **docs/user-guide/getting-started.md** - User getting started guide
   - Completely rewritten for OpenShift deployment
   - Removed outdated LlamaDeploy/Python references
   - Added OpenShift-specific prerequisites
   - Updated deployment steps with oc commands
   - Added project creation and runner secrets configuration
   - Updated troubleshooting for common OpenShift issues

3. **docs/index.md** - Documentation landing page
   - Rewrote to reflect current architecture
   - Added clear feature list (agentic sessions, RFE workflows, multi-tenancy)
   - Updated component descriptions
   - Improved navigation to other docs
   - Added integration details (GitHub App, OAuth, Jira)

### Key Changes

- **Architecture**: Updated all references from "Kubernetes-native" to "OpenShift-native"
- **Components**: Accurately described all five components (Frontend, Backend, Operator, Runner, Content Service)
- **Deployment**: Changed from local Python development to OpenShift deployment with make/oc
- **Prerequisites**: Updated from Python/Node tools to OpenShift cluster/oc CLI
- **Terminology**: Consistent use of "agentic sessions", "RFE workflows", "projects"
- **Commands**: All examples now use oc commands and make targets
- **Troubleshooting**: Updated for OpenShift-specific issues (pods, jobs, WebSockets)

### Validation

All documentation updates have been validated to ensure:
- Technical accuracy matches codebase (verified against components/ source)
- Commands reference actual Makefile targets and scripts
- Links point to existing files
- Terminology is consistent throughout
- Examples are realistic and achievable
