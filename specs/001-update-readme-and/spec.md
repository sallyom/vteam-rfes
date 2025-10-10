# Feature Specification: Update README and Other Docs for vTeam

**Feature Branch**: `001-update-readme-and`
**Created**: 2025-10-10
**Status**: Draft
**Input**: User description: "Update README and other docs for vTeam. Use components code to inform."

## Execution Flow (main)
```
1. Parse user description from Input
   → Feature: Documentation updates for vTeam platform
2. Extract key concepts from description
   → Actors: developers, DevOps engineers, platform operators
   → Actions: update, create, reference, document
   → Data: architecture, deployment instructions, configuration
   → Constraints: must reflect current codebase implementation
3. For each unclear aspect:
   → All requirements clearly specified in rfe.md
4. Fill User Scenarios & Testing section
   → Three primary user journeys identified
5. Generate Functional Requirements
   → All requirements are testable and derived from acceptance criteria
6. Identify Key Entities (if data involved)
   → Documentation files and deployment artifacts
7. Run Review Checklist
   → No implementation details, spec focused on what needs documenting
8. Return: SUCCESS (spec ready for planning)
```

---

## ⚡ Quick Guidelines
- ✅ Focus on WHAT users need and WHY
- ❌ Avoid HOW to implement (no tech stack, APIs, code structure)
- 👥 Written for business stakeholders, not developers

---

## User Scenarios & Testing *(mandatory)*

### Primary User Story

A developer new to vTeam needs to set up a local development environment. They read the getting-started.md guide, install OpenShift Local (CRC), clone the repository, configure the .env file with their Anthropic API key, run `make dev-start`, and successfully access the vTeam UI. They then create their first project and agentic session with git repository access.

A DevOps engineer needs to deploy vTeam to a production OpenShift cluster. They log into the cluster, prepare the .env configuration file, run the `./deploy.sh` script from the manifests directory, verify that all pods are running, access the frontend via the OpenShift route, create project namespaces, and configure git authentication secrets per project so users can run sessions with repository operations.

A platform operator needs to enable git operations (clone, commit, push) for sessions in a specific project. They read the git authentication documentation, create a GitHub personal access token, create a Kubernetes secret in the project namespace containing git credentials, reference the secret in ProjectSettings via the UI, and verify that sessions can now perform git operations.

### Acceptance Scenarios

1. **Given** a developer with no prior vTeam knowledge, **When** they follow the getting-started.md guide, **Then** they successfully set up a local development environment and create their first session within 30 minutes
2. **Given** a DevOps engineer with OpenShift cluster access, **When** they follow the deployment instructions in README.md and OPENSHIFT_DEPLOY.md, **Then** they successfully deploy vTeam to production with all components running
3. **Given** documentation that references the deploy.sh script, **When** users read the deployment options, **Then** they understand how to deploy with standard settings, custom namespace, secrets-only mode, and uninstall
4. **Given** multiple documentation files (README.md, components/README.md, getting-started.md, OPENSHIFT_DEPLOY.md), **When** users follow cross-references between documents, **Then** all paths are accurate and consistent
5. **Given** a platform operator configuring git authentication, **When** they follow GIT_AUTH_SETUP.md and related documentation, **Then** they successfully create project-level secrets and enable git operations for sessions
6. **Given** the architecture section in README.md, **When** developers review it, **Then** it accurately describes Next.js 15 frontend, Go backend (Gin framework), Kubernetes operator, and Claude Code SDK runner
7. **Given** the troubleshooting section in getting-started.md, **When** users encounter common issues (pods not starting, session failures, WebSocket errors), **Then** they find actionable solutions

### Edge Cases

- What happens when users follow outdated deployment instructions that don't reflect the current deploy.sh script options?
  - Documentation must be synchronized with actual script capabilities
- How does the system handle users who skip git authentication configuration?
  - Documentation must clearly state that git operations require separate secret creation per project
- What happens when cross-references between documents use incorrect paths?
  - All links must be validated to ensure they point to existing files
- How do users troubleshoot issues if the troubleshooting section doesn't cover their specific error?
  - Documentation should link to GitHub issues for reporting new problems

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: README.md MUST accurately describe the current vTeam architecture including Next.js 15 frontend (React + Shadcn UI), Go backend (Gin framework), Kubernetes operator, and Claude Code SDK runner
- **FR-002**: README.md MUST include a component table showing accurate technology stack for each component
- **FR-003**: README.md MUST reference the `./deploy.sh` script in the manifests directory for deployment
- **FR-004**: README.md MUST clearly state git authentication requirements with a link to GIT_AUTH_SETUP.md
- **FR-005**: components/README.md MUST show the current directory structure (frontend/, backend/, operator/, runners/, manifests/, scripts/)
- **FR-006**: components/README.md MUST include the agentic session flow with 7 steps from user creation to result storage
- **FR-007**: components/README.md MUST provide quick start sections covering production deployment, local development, and custom image builds
- **FR-008**: getting-started.md MUST list all prerequisites including oc CLI, kustomize, OpenShift Local (CRC), and Anthropic API key
- **FR-009**: getting-started.md MUST document local development setup including CRC installation and configuration
- **FR-010**: getting-started.md MUST cover production deployment workflow from login to verification
- **FR-011**: getting-started.md MUST document git authentication configuration with example commands
- **FR-012**: getting-started.md MUST include post-deployment steps covering project creation, runner secrets, and first session
- **FR-013**: getting-started.md MUST provide troubleshooting guidance for common issues (pods not starting, session failures, WebSocket issues)
- **FR-014**: OPENSHIFT_DEPLOY.md MUST list prerequisites explicitly (oc CLI, kustomize, registry access)
- **FR-015**: OPENSHIFT_DEPLOY.md MUST document deployment script options (standard deploy, custom namespace, secrets-only, uninstall)
- **FR-016**: OPENSHIFT_DEPLOY.md MUST emphasize that git authentication secrets must be created separately per project with link to GIT_AUTH_SETUP.md
- **FR-017**: OPENSHIFT_DEPLOY.md MUST document post-deployment setup covering runner secrets, project creation, and git configuration
- **FR-018**: All documentation MUST use consistent namespace references (ambient-code by default)
- **FR-019**: All documentation MUST use consistent image registry references (quay.io/ambient_code for pre-built images)
- **FR-020**: All cross-references between documents MUST use correct relative paths that resolve to existing files
- **FR-021**: All code blocks in documentation MUST be properly formatted with language-specific syntax highlighting
- **FR-022**: Documentation MUST be self-contained and not assume prior OpenShift/Kubernetes knowledge
- **FR-023**: Documentation MUST provide progressive disclosure (quick start first, detailed configuration later)

### Key Entities *(include if feature involves data)*

- **README.md**: Root-level documentation file providing platform overview, architecture description, quick start guide, and links to detailed documentation
- **components/README.md**: Component-specific overview showing directory structure, architecture flow, and quick start commands
- **getting-started.md**: Comprehensive guide covering prerequisites, local development setup, production deployment, and troubleshooting
- **OPENSHIFT_DEPLOY.md**: Deployment-focused guide documenting deploy.sh script usage, options, and post-deployment configuration
- **GIT_AUTH_SETUP.md**: Git authentication configuration guide referenced by multiple documents
- **Deploy Script (deploy.sh)**: Deployment automation script whose capabilities must be accurately documented
- **Component Directories**: Source code directories (frontend/, backend/, operator/, runners/) that inform accurate documentation

---

## Review & Acceptance Checklist
*GATE: Automated checks run during main() execution*

### Content Quality
- [x] No implementation details (languages, frameworks, APIs) - Spec focuses on documentation content, not implementation
- [x] Focused on user value and business needs - Documentation enables successful deployment and usage
- [x] Written for non-technical stakeholders - Spec describes user outcomes and capabilities
- [x] All mandatory sections completed - All sections filled with concrete requirements

### Requirement Completeness
- [x] No [NEEDS CLARIFICATION] markers remain - All requirements derived from rfe.md
- [x] Requirements are testable and unambiguous - Each requirement can be verified by inspecting documentation
- [x] Success criteria are measurable - Acceptance criteria specify observable outcomes
- [x] Scope is clearly bounded - Out-of-scope items defined in rfe.md (no code changes, no UI changes, no video tutorials)
- [x] Dependencies and assumptions identified - Assumes existing codebase structure; depends on GIT_AUTH_SETUP.md

---

## Execution Status
*Updated by main() during processing*

- [x] User description parsed
- [x] Key concepts extracted
- [x] Ambiguities marked (none identified)
- [x] User scenarios defined
- [x] Requirements generated
- [x] Entities identified
- [x] Review checklist passed

---
