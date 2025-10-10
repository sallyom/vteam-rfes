# Documentation Updates for vTeam

**Feature Overview:**
*Update vTeam documentation to accurately reflect the current codebase implementation, ensure all instructions are straightforward and concise, and provide comprehensive guides for local development and OpenShift deployment. The documentation should serve as the single source of truth for developers and operators deploying and using the vTeam platform.*

## Goals

*Provide accurate, concise, and up-to-date documentation that enables users to:*

- Quickly understand the vTeam architecture and component structure
- Successfully deploy vTeam to OpenShift clusters using the `components/manifests/deploy.sh` script
- Set up local development environments with OpenShift Local (CRC)
- Configure git authentication separately per project (GitHub secrets)
- Build custom container images and deploy with custom registries
- Troubleshoot common deployment and runtime issues

*The updated documentation benefits developers, DevOps engineers, and platform operators who need to deploy, configure, and maintain vTeam instances. This improves onboarding time, reduces support burden, and ensures users can successfully leverage the platform's capabilities.*

## Out of Scope

*The following items are explicitly out of scope for this documentation update:*

- Code changes or refactoring to the vTeam components themselves
- Changes to the deployment scripts or Kubernetes manifests
- Implementation of new features or capabilities
- UI/UX changes to the frontend application
- Performance optimization or architectural changes
- Creation of video tutorials or interactive training materials

## Requirements

*The documentation update must deliver the following capabilities:*

- **[MVP] Update README.md** - Reflect current architecture with accurate component descriptions (Next.js 15, Go backend, operator, Claude Code SDK runner)
- **[MVP] Update components/README.md** - Provide concise component overview and quick start instructions
- **[MVP] Create getting-started.md** - Comprehensive guide covering prerequisites, local development setup, production deployment, and post-deployment configuration
- **[MVP] Update OPENSHIFT_DEPLOY.md** - Document deploy.sh script usage, options (standard deploy, custom namespace, secrets-only, uninstall), and emphasize separate GitHub secret creation
- **[MVP] Reference GIT_AUTH_SETUP.md** - Ensure all relevant docs link to git authentication setup guide
- **Non-MVP** Update component-specific README files in frontend/, backend/, operator/, runners/
- **Non-MVP** Create architecture diagrams showing component interaction and data flow
- **Non-MVP** Expand troubleshooting sections with common error scenarios

## Done - Acceptance Criteria

*The documentation update is complete when:*

1. **README.md accurately describes the current system**
   - Architecture section reflects Go backend (Gin framework), Next.js 15 frontend (React + Shadcn UI), Kubernetes operator, and Claude Code SDK runner
   - Component table shows accurate technology stack
   - Deployment instructions reference `./deploy.sh` script in manifests directory
   - Git authentication requirements are clearly stated with link to GIT_AUTH_SETUP.md

2. **components/README.md provides clear component overview**
   - Directory structure shows current layout (frontend/, backend/, operator/, runners/, manifests/, scripts/)
   - Architecture section includes agentic session flow (7 steps from user creation to result storage)
   - Quick start sections cover production deployment, local development, and custom image builds
   - All paths and script references are accurate

3. **getting-started.md guide is comprehensive**
   - Prerequisites section lists all required tools and API keys
   - Local development setup includes CRC installation and configuration
   - Production deployment section covers full workflow from login to verification
   - Git authentication configuration is documented with example commands
   - Post-deployment section covers project creation, runner secrets, and first session
   - Troubleshooting section addresses common issues (pods not starting, session failures, WebSocket issues)

4. **OPENSHIFT_DEPLOY.md is updated with deploy.sh details**
   - Prerequisites explicitly list oc CLI, kustomize, and registry access
   - Quick deploy section shows 4-step process (configure, deploy, verify, access)
   - Deployment script options are documented (standard, custom namespace, secrets-only, uninstall)
   - Git authentication section emphasizes separate secret creation with link to GIT_AUTH_SETUP.md
   - Post-deployment setup covers runner secrets, project creation, and git setup
   - Cleanup section documents uninstall commands

5. **All documentation is internally consistent**
   - Cross-references between documents use correct paths
   - Command examples use consistent namespace (ambient-code by default)
   - Image registry references use quay.io/ambient_code for pre-built images
   - All code blocks are properly formatted and tested

## Use Cases - i.e. User Experience & Workflow

*Primary use cases and user workflows:*

### Use Case 1: First-Time Local Development Setup

**Actor:** Developer new to vTeam
**Goal:** Set up local development environment and run vTeam on OpenShift Local

**Main Success Scenario:**
1. Developer reads getting-started.md prerequisites section
2. Developer installs OpenShift Local (CRC) and obtains pull secret
3. Developer clones vTeam repository and configures .env file
4. Developer runs `make dev-start` from project root
5. System builds images, deploys components, and configures routes
6. Developer accesses UI at localhost:3000 or route URL
7. Developer creates first project and agentic session

**Alternative Flow - Missing Prerequisites:**
- If CRC not installed → Documentation provides installation command
- If pull secret missing → Documentation links to console.redhat.com
- If ANTHROPIC_API_KEY not set → Deployment succeeds but sessions fail with clear error

### Use Case 2: Production Deployment to OpenShift

**Actor:** DevOps Engineer
**Goal:** Deploy vTeam to production OpenShift cluster

**Main Success Scenario:**
1. Engineer logs into OpenShift cluster with admin credentials
2. Engineer prepares .env file with ANTHROPIC_API_KEY and optional GitHub App credentials
3. Engineer runs `./deploy.sh` from components/manifests directory
4. System deploys backend, frontend, operator, CRDs, and RBAC
5. Engineer verifies pods are running via `oc get pods -n ambient-code`
6. Engineer accesses frontend via OpenShift route
7. Engineer creates project namespaces via UI
8. Engineer creates Kubernetes secrets with git credentials per project
9. Users can now create sessions with git repository access

**Alternative Flow - Custom Namespace:**
- Engineer runs `NAMESPACE=my-vteam ./deploy.sh`
- System deploys to specified namespace instead of default

**Alternative Flow - Custom Images:**
- Engineer builds images with `make build-all REGISTRY=quay.io/myorg`
- Engineer pushes images with `make push-all REGISTRY=quay.io/myorg`
- Engineer deploys with `CONTAINER_REGISTRY=quay.io/myorg ./deploy.sh`

### Use Case 3: Configuring Git Authentication

**Actor:** Platform Operator
**Goal:** Enable git operations (clone, commit, push) for sessions

**Main Success Scenario:**
1. Operator reads GIT_AUTH_SETUP.md linked from deployment guide
2. Operator creates GitHub personal access token with repo permissions
3. Operator creates Kubernetes secret in project namespace with GIT_TOKEN, GIT_USER_NAME, GIT_USER_EMAIL
4. Operator references secret name in ProjectSettings via UI (Settings → Runner Secrets)
5. Users create sessions with HTTPS git URLs
6. Runner automatically authenticates using GIT_TOKEN from project secret

**Alternative Flow - GitHub App Authentication:**
- Admin configures GitHub App credentials in platform namespace
- Users link GitHub accounts via OAuth
- Both project-level secrets and GitHub App tokens work together

## Documentation Considerations

*Documentation planning and customer needs:*

### Structure and Organization
- Primary README.md remains in project root
- Detailed guides live in docs/ directory (getting-started.md, OPENSHIFT_DEPLOY.md, etc.)
- Component-specific documentation in respective component directories
- Cross-references use relative paths for repository portability

### Maintenance Strategy
- Documentation should be updated whenever component versions change (e.g., Next.js upgrade)
- Deploy script changes must be reflected in OPENSHIFT_DEPLOY.md
- New configuration options should be added to getting-started.md
- Troubleshooting section should grow based on user-reported issues

### Format and Style
- Use clear, concise language suitable for technical audience
- Provide code examples with comments explaining intent
- Use consistent command formatting (bash code blocks)
- Include expected output where helpful for verification
- Use bold for important notes and warnings

### Accessibility
- All documentation in markdown format for easy reading on GitHub and locally
- No reliance on external documentation hosting (can be published to MkDocs later)
- Self-contained instructions that don't assume prior OpenShift/Kubernetes knowledge
- Progressive disclosure: quick start first, detailed configuration later

## Questions to answer

*Refinement and architectural questions:*

1. **Should we include a video walkthrough or screenshots for UI operations?**
   - Out of scope for this RFE; text documentation sufficient for MVP
   - Can be added as enhancement later

2. **Do we need separate guides for different OpenShift versions (4.12, 4.13, 4.14)?**
   - No; deploy.sh and manifests are version-agnostic
   - Prerequisites should note "OpenShift 4.x" without specific version

3. **Should troubleshooting guide include solutions for every possible error?**
   - No; focus on most common issues (pods not starting, session failures, auth errors)
   - Link to GitHub issues for reporting new problems

4. **How detailed should the git authentication documentation be?**
   - Provide clear examples for common scenarios (GitHub personal access token)
   - Reference GIT_AUTH_SETUP.md for comprehensive coverage
   - Document authentication priority (GIT_TOKEN > GITHUB_TOKEN > none)

5. **Should we document the internal architecture (CR lifecycle, operator reconciliation loops)?**
   - Brief overview in main README sufficient for this RFE
   - Detailed architecture documentation can be separate doc (out of scope for this update)

## Background & Strategic Fit

*Context and strategic alignment:*

vTeam is positioned as an OpenShift-native AI automation platform for enterprise use. Accurate, comprehensive documentation is critical for:

- **Adoption:** Potential users need to quickly understand what vTeam is and how to deploy it
- **Onboarding:** New team members must be able to set up development environments independently
- **Operations:** Platform teams require clear deployment and configuration guidance
- **Support:** Good documentation reduces support burden and empowers users to self-serve

The current documentation is partially outdated and lacks clarity around deployment requirements (especially git authentication). This update aligns with the strategic goal of making vTeam production-ready for enterprise deployments.

## Customer Considerations

*Customer-specific considerations for documentation:*

### Enterprise Customers
- Require clear security model documentation (RBAC, secret management, namespace isolation)
- Need production deployment guide with high availability and scalability considerations
- Value troubleshooting guides for common operational issues

### Open Source Community
- Benefit from quick start guides for local development
- Appreciate example configurations and common use cases
- Need clear contribution guidelines (out of scope for this RFE)

### Internal Development Teams
- Require accurate technical details about component architecture
- Need build and deployment instructions for custom images
- Value debugging and development workflow documentation

### Deployment Considerations
- Documentation must work for both connected and air-gapped environments (note prerequisites)
- Should support multiple deployment scenarios (development, staging, production)
- Must clarify separation between platform-level config (deploy.sh) and project-level config (git secrets)
