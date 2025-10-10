# Documentation Contract: README.md

**Target File**: `/workspace/.../vTeam/README.md`
**Type**: Root-level overview documentation
**Status**: Update required

---

## Required Sections

### 1. Overview
**Purpose**: Brief introduction to vTeam platform
**Requirements**: FR-001 (architecture description)
**Content**:
- 1-2 paragraph description of vTeam as an OpenShift-native AI automation platform
- Key capabilities (agentic sessions, git integration, Claude Code SDK)
- Target audience (developers, DevOps engineers, platform operators)

**Success Criteria**:
- [ ] Concise description (<200 words)
- [ ] Mentions OpenShift/Kubernetes as target platform
- [ ] Highlights Claude Code SDK runner as core component

---

### 2. Architecture
**Purpose**: Describe component architecture and technology stack
**Requirements**: FR-001, FR-002
**Content**:
- Component table with columns: Component | Technology | Description
- Accurate version numbers (Next.js 15, Go 1.24, etc.)
- Brief description of each component's role

**Component Table Schema**:
```markdown
| Component | Technology | Description |
|-----------|------------|-------------|
| Frontend | Next.js 15.5.2, React 19.1.0, Shadcn UI | User interface for project and session management |
| Backend | Go 1.24, Gin framework | REST API and WebSocket server for session coordination |
| Operator | Kubernetes Operator | Manages custom resources (AgenticSession, Project, RunnerSecret) |
| Runner | Claude Code SDK | Executes AI-powered development sessions with tool access |
```

**Success Criteria**:
- [ ] All four components listed with accurate technologies
- [ ] Version numbers match package.json and go.mod
- [ ] Description is clear and concise for each component

---

### 3. Quick Start
**Purpose**: Provide rapid deployment instructions with link to detailed guide
**Requirements**: FR-003 (reference deploy.sh)
**Content**:
- Prerequisites (OpenShift cluster access, ANTHROPIC_API_KEY)
- Quick deploy command referencing `./deploy.sh` in manifests directory
- Link to getting-started.md for comprehensive setup

**Code Block Example**:
```markdown
## Quick Start

Prerequisites:
- OpenShift cluster with admin access
- `oc` CLI tool installed
- Anthropic API key

Deploy to OpenShift:
```bash
cd components/manifests
# Configure your API key
export ANTHROPIC_API_KEY="your-key-here"
./deploy.sh
```

For detailed setup instructions, see [getting-started.md](docs/getting-started.md).
```

**Success Criteria**:
- [ ] References deploy.sh script with correct path
- [ ] Lists essential prerequisites only
- [ ] Links to getting-started.md for comprehensive guide

---

### 4. Git Authentication
**Purpose**: Explain git authentication requirements and link to setup guide
**Requirements**: FR-004
**Content**:
- Statement that git operations require separate secret configuration
- Link to GIT_AUTH_SETUP.md
- Brief explanation of project-level vs. platform-level authentication

**Example Content**:
```markdown
## Git Authentication

To enable git operations (clone, commit, push) in agentic sessions, you must configure git credentials per project:

- **Project-level secrets**: Create Kubernetes secrets in each project namespace
- **GitHub App (optional)**: Configure GitHub App for OAuth authentication

See [GIT_AUTH_SETUP.md](docs/GIT_AUTH_SETUP.md) for detailed configuration instructions.
```

**Success Criteria**:
- [ ] Clearly states git auth is NOT automatic
- [ ] Links to GIT_AUTH_SETUP.md with correct path
- [ ] Mentions project-level secret requirement

---

### 5. Additional Sections (Optional)
- Development (link to components/README.md)
- Contributing (out of scope for this RFE)
- License

---

## Cross-Reference Requirements

**Outgoing Links**:
- `docs/getting-started.md` (comprehensive setup guide)
- `docs/OPENSHIFT_DEPLOY.md` (deployment details)
- `docs/GIT_AUTH_SETUP.md` (git configuration)
- `components/README.md` (component-specific development)

**Link Validation**:
- [ ] All links use relative paths from repository root
- [ ] All target files exist or will be created by this feature
- [ ] Links render correctly in GitHub markdown viewer

---

## Code Block Requirements

**Language Identifiers Required**:
- `bash` for shell commands
- `yaml` for Kubernetes manifests
- `markdown` for meta-examples

**Standards**:
- [ ] All code blocks specify language for syntax highlighting
- [ ] Bash commands use consistent namespace (ambient-code)
- [ ] No placeholder values in code blocks (use environment variables)

---

## Consistency Requirements (FR-018, FR-019)

**Namespace References**:
- [ ] Default namespace is `ambient-code` in all examples
- [ ] Document NAMESPACE override: `NAMESPACE=custom ./deploy.sh`

**Registry References**:
- [ ] Pre-built images reference `quay.io/ambient_code`
- [ ] Custom registry override documented: `CONTAINER_REGISTRY=quay.io/myorg`

---

## Validation Checklist

Before marking this contract as complete:
- [ ] All required sections present with non-empty content
- [ ] Component table matches actual source code (package.json, go.mod)
- [ ] Deploy script path is correct (components/manifests/deploy.sh)
- [ ] All cross-references use valid paths
- [ ] Code blocks have language identifiers
- [ ] Consistent namespace and registry references
- [ ] Functional requirements FR-001 through FR-004 fully addressed

---

**Contract Version**: 1.0
**Generated**: 2025-10-10 by /plan command execution
