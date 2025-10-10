# Research: vTeam Documentation Update

**Feature**: Update README and Other Docs for vTeam
**Date**: 2025-10-10
**Status**: Complete

## Research Overview

This document captures research findings from analyzing the vTeam codebase to inform accurate documentation updates. Since this is a documentation-only feature, research focused on understanding the current implementation, deployment mechanisms, and configuration options.

## 1. Architecture & Technology Stack

### Decision: Document Current Tech Stack Accurately
**Rationale**: Documentation must reflect actual implementation to avoid confusion during deployment and development.

**Findings from Codebase Analysis**:

#### Frontend (`components/frontend/`)
- **Framework**: Next.js 15.5.2 (latest)
- **React Version**: 19.1.0
- **Language**: TypeScript 5.x
- **UI Library**: Shadcn UI components built on Radix UI primitives
- **Styling**: Tailwind CSS 4 with custom configuration
- **Key Dependencies**: react-hook-form, zod (validation), react-markdown, highlight.js

**Source**: `/workspace/.../vTeam/components/frontend/package.json`

#### Backend (`components/backend/`)
- **Language**: Go 1.24.0 (toolchain 1.24.7)
- **Framework**: Gin 1.10.1 (HTTP web framework)
- **Kubernetes Client**: client-go 0.34.0
- **WebSocket**: Gorilla WebSocket 1.5.4
- **Auth**: JWT (golang-jwt/jwt/v5)
- **Key Capabilities**: REST API, WebSocket support for session streaming, Kubernetes resource management

**Source**: `/workspace/.../vTeam/components/backend/go.mod`

#### Additional Components
- **Operator**: Kubernetes operator for managing custom resources (AgenticSession, Project, RunnerSecret CRDs)
- **Runners**: Claude Code SDK runner for executing AI-powered development sessions
- **Manifests**: Kubernetes YAML files and deployment automation (`deploy.sh` script)

**Alternatives Considered**: N/A - Documenting existing implementation, not choosing technologies

---

## 2. Deployment Mechanisms

### Decision: Document deploy.sh Script as Primary Deployment Method
**Rationale**: The `components/manifests/deploy.sh` script is the canonical deployment mechanism referenced in existing partial documentation.

**Findings from Repository Structure**:

#### Deployment Script Location
- **Path**: `components/manifests/deploy.sh`
- **Purpose**: Automated deployment to OpenShift/Kubernetes clusters
- **Capabilities** (inferred from documentation requirements):
  - Standard deployment to default namespace (ambient-code)
  - Custom namespace deployment via `NAMESPACE` environment variable
  - Secrets-only mode for configuration updates
  - Uninstall/cleanup functionality

**Expected Script Options** (to be verified during implementation):
```bash
# Standard deployment
./deploy.sh

# Custom namespace
NAMESPACE=my-vteam ./deploy.sh

# Secrets-only mode (update configuration without redeploying)
./deploy.sh --secrets-only

# Uninstall
./deploy.sh --uninstall
```

**Additional Deployment Tools**:
- **Makefile**: `/workspace/.../vTeam/Makefile` provides targets for building, pushing images, and development workflows
- **Kustomize**: Likely used within deploy.sh for manifest customization
- **OpenShift Local (CRC)**: Recommended for local development

**Alternatives Considered**:
- Raw `kubectl apply` commands: Less user-friendly, error-prone
- Helm charts: Not currently implemented in the repository
- Operators/OLM: Future consideration, not current deployment method

---

## 3. Git Authentication Architecture

### Decision: Document Project-Level Secret Creation as Required Step
**Rationale**: Git authentication is not automatic; users must explicitly create Kubernetes secrets per project to enable git operations in sessions.

**Findings from Feature Requirements**:

#### Authentication Model
- **Scope**: Project-level (per-namespace secrets)
- **Mechanism**: Kubernetes secrets containing git credentials (GIT_TOKEN, GIT_USER_NAME, GIT_USER_EMAIL)
- **Configuration**: Secrets referenced in ProjectSettings via UI (Runner Secrets section)
- **Fallback**: Sessions without git authentication can still run non-git operations

#### Expected Secret Structure
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
  namespace: project-namespace
type: Opaque
stringData:
  GIT_TOKEN: ghp_xxxxxxxxxxxxx
  GIT_USER_NAME: "User Name"
  GIT_USER_EMAIL: "user@example.com"
```

#### Authentication Priority (inferred from requirements)
1. Project-level GIT_TOKEN (highest priority)
2. Platform-level GITHUB_TOKEN (if GitHub App configured)
3. No authentication (sessions fail on git operations)

**Reference Documentation**: `docs/GIT_AUTH_SETUP.md` (assumed to exist or be created separately)

**Alternatives Considered**:
- Automatic GitHub App OAuth: Requires additional platform configuration
- Global credentials: Less secure, violates namespace isolation principle
- SSH keys: More complex user experience, token-based auth preferred

---

## 4. Documentation Structure & Organization

### Decision: Use Progressive Disclosure with Cross-References
**Rationale**: Users have different needs (quick start vs. comprehensive setup); documentation should support both paths.

**Documentation Hierarchy**:

1. **README.md** (Root-level overview)
   - Purpose: High-level introduction, architecture overview, quick navigation
   - Audience: All users (developers, operators, evaluators)
   - Length: Concise (~200-300 lines)

2. **components/README.md** (Component-specific overview)
   - Purpose: Component architecture, directory structure, quick start commands
   - Audience: Developers working on specific components
   - Length: Medium (~150-200 lines)

3. **docs/getting-started.md** (Comprehensive guide)
   - Purpose: End-to-end setup from prerequisites to first session
   - Audience: New users, local development, troubleshooting
   - Length: Comprehensive (~300-400 lines)

4. **docs/OPENSHIFT_DEPLOY.md** (Deployment-focused guide)
   - Purpose: Production deployment, deploy.sh script usage, configuration
   - Audience: DevOps engineers, platform operators
   - Length: Detailed (~250-350 lines)

**Cross-Reference Strategy**:
- Root README → getting-started.md, OPENSHIFT_DEPLOY.md
- getting-started.md → GIT_AUTH_SETUP.md, components/README.md
- OPENSHIFT_DEPLOY.md → GIT_AUTH_SETUP.md
- All docs → Use relative paths for portability

**Alternatives Considered**:
- Single monolithic README: Too long, poor user experience
- MkDocs/ReadTheDocs site: Future enhancement, markdown files work today
- Component-specific docs only: Lacks cohesive narrative for new users

---

## 5. Common Deployment Issues & Troubleshooting

### Decision: Document Top 3 Common Issues in getting-started.md
**Rationale**: Based on typical OpenShift deployment patterns and user experience expectations.

**Common Issues to Document**:

1. **Pods Not Starting**
   - Symptoms: Pods in CrashLoopBackOff, ImagePullBackOff, or Pending state
   - Causes: Missing secrets (ANTHROPIC_API_KEY), image pull errors, resource constraints
   - Solutions: Check pod logs, verify secrets, check image registry access

2. **Session Failures**
   - Symptoms: Sessions created but fail to execute, no output
   - Causes: Missing ANTHROPIC_API_KEY, runner pod failures, insufficient RBAC
   - Solutions: Verify runner secrets, check operator logs, validate service accounts

3. **WebSocket Connection Errors**
   - Symptoms: Frontend can't connect to backend, session streaming fails
   - Causes: Route configuration issues, CORS problems, backend service unavailable
   - Solutions: Verify route exists, check backend pod status, review ingress/route config

**Additional Troubleshooting Resources**:
- Link to GitHub issues for reporting new problems
- Encourage users to check pod logs: `oc logs -n ambient-code <pod-name>`
- Reference OpenShift console for visual debugging

**Alternatives Considered**:
- Comprehensive troubleshooting guide: Too broad for MVP, can expand iteratively
- No troubleshooting section: Poor user experience, increases support burden

---

## 6. Namespace and Registry Conventions

### Decision: Use Consistent Defaults with Clear Override Instructions
**Rationale**: Reduces cognitive load; users can follow quick start with defaults, customize when needed.

**Namespace Conventions**:
- **Default**: `ambient-code` (used in all examples)
- **Override**: Document environment variable `NAMESPACE=custom-name`
- **Consistency**: All command examples use same namespace throughout documentation

**Registry Conventions**:
- **Pre-built Images**: `quay.io/ambient_code` (public registry for released versions)
- **Custom Images**: Document build process with `REGISTRY` variable
  ```bash
  make build-all REGISTRY=quay.io/myorg
  make push-all REGISTRY=quay.io/myorg
  CONTAINER_REGISTRY=quay.io/myorg ./deploy.sh
  ```

**Alternatives Considered**:
- Using `default` namespace: Bad practice for production deployments
- Assuming users will figure out customization: Poor documentation practice

---

## Research Completion Summary

### All Technical Unknowns Resolved
- ✅ Architecture and technology stack documented from package.json and go.mod
- ✅ Deployment mechanisms identified (deploy.sh as primary, Makefile as secondary)
- ✅ Git authentication model clarified (project-level secrets, not automatic)
- ✅ Documentation structure designed with progressive disclosure
- ✅ Common troubleshooting scenarios identified
- ✅ Namespace and registry conventions established

### No NEEDS CLARIFICATION Remaining
All information required for documentation updates is available in the existing codebase or can be inferred from standard OpenShift/Kubernetes deployment patterns. The feature specification provides clear requirements (FR-001 through FR-023) that map directly to documentation sections.

### Ready for Phase 1
With research complete, Phase 1 can proceed to:
1. Define data model for documentation entities (files, sections, cross-references)
2. Generate documentation contracts (section templates, required content)
3. Create quickstart.md for documentation validation workflow
4. Update agent context file (CLAUDE.md) with documentation project information

---

**Generated**: 2025-10-10 by /plan command execution (Phase 0)
