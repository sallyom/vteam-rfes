# Documentation Contract: getting-started.md

**Target File**: `/workspace/.../vTeam/docs/getting-started.md`
**Type**: Comprehensive setup guide
**Status**: Creation required

---

## Required Sections

### 1. Prerequisites
**Purpose**: List all required tools, access, and credentials
**Requirements**: FR-008
**Content**:
- CLI tools (oc, kustomize)
- OpenShift cluster access or OpenShift Local (CRC) for local dev
- Anthropic API key (with link to obtain)
- Optional: GitHub personal access token for git operations

**Code Block Example**:
```markdown
## Prerequisites

### Required Tools
- **OpenShift CLI (`oc`)**: [Install instructions](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html)
- **Kustomize**: [Install instructions](https://kubectl.docs.kubernetes.io/installation/kustomize/)

### Cluster Access
- OpenShift cluster with admin privileges, OR
- OpenShift Local (CRC) for local development

### API Keys
- **Anthropic API Key**: Obtain from [Anthropic Console](https://console.anthropic.com/)
- **GitHub Token** (optional): For git operations in sessions
```

**Success Criteria**:
- [ ] All required tools explicitly listed with install links
- [ ] Both cluster options documented (production + local dev)
- [ ] API key requirements clearly stated
- [ ] Links to external resources are valid

---

### 2. Local Development Setup
**Purpose**: Guide users through CRC installation and local deployment
**Requirements**: FR-009
**Content**:
- CRC installation instructions
- CRC configuration (memory, CPU recommendations)
- Pull secret configuration
- Local deployment commands

**Code Block Example**:
```markdown
## Local Development Setup

### Install OpenShift Local (CRC)

1. Download CRC from [Red Hat Console](https://console.redhat.com/openshift/create/local)
2. Extract and setup:
   ```bash
   tar -xf crc-linux-amd64.tar.xz
   cd crc-linux-amd64-*
   ./crc setup
   ```

3. Configure resources (recommended):
   ```bash
   ./crc config set memory 16384
   ./crc config set cpus 6
   ```

4. Start CRC with pull secret:
   ```bash
   ./crc start -p /path/to/pull-secret.txt
   ```

5. Login to cluster:
   ```bash
   eval $(./crc oc-env)
   oc login -u kubeadmin https://api.crc.testing:6443
   ```

### Deploy vTeam Locally

1. Clone repository:
   ```bash
   git clone https://github.com/your-org/vTeam.git
   cd vTeam
   ```

2. Configure API key:
   ```bash
   cd components/manifests
   cp .env.example .env
   # Edit .env and add your ANTHROPIC_API_KEY
   ```

3. Deploy:
   ```bash
   ./deploy.sh
   ```

4. Verify deployment:
   ```bash
   oc get pods -n ambient-code
   ```
```

**Success Criteria**:
- [ ] CRC installation steps are clear and complete
- [ ] Resource recommendations provided (memory, CPU)
- [ ] Pull secret configuration explained
- [ ] Local deployment workflow is end-to-end
- [ ] Verification steps included

---

### 3. Production Deployment
**Purpose**: Guide DevOps engineers through production deployment workflow
**Requirements**: FR-010
**Content**:
- Login to production cluster
- Environment configuration (.env file)
- Deploy script execution
- Pod verification
- Route access

**Code Block Example**:
```markdown
## Production Deployment

### 1. Login to Cluster
```bash
oc login https://api.cluster.example.com:6443 --token=your-token
```

### 2. Configure Environment
```bash
cd components/manifests
cat > .env << EOF
ANTHROPIC_API_KEY=your-api-key-here
NAMESPACE=ambient-code
EOF
```

### 3. Deploy vTeam
```bash
./deploy.sh
```

### 4. Verify Deployment
```bash
# Check all pods are running
oc get pods -n ambient-code

# Expected output: All pods in Running state
# - frontend-xxx
# - backend-xxx
# - operator-xxx
```

### 5. Access Frontend
```bash
# Get the frontend route
oc get route frontend -n ambient-code

# Access the URL in your browser
```
```

**Success Criteria**:
- [ ] Production login flow documented
- [ ] Environment configuration clearly explained
- [ ] Deploy script usage shown
- [ ] Verification steps with expected output
- [ ] Route access instructions included

---

### 4. Git Authentication Configuration
**Purpose**: Explain how to enable git operations in sessions
**Requirements**: FR-011
**Content**:
- Secret creation command with example values
- Reference to GIT_AUTH_SETUP.md for comprehensive guide
- How to reference secret in ProjectSettings

**Code Block Example**:
```markdown
## Git Authentication Configuration

To enable git operations (clone, commit, push) in agentic sessions, create a Kubernetes secret in each project namespace:

```bash
# Create git credentials secret
oc create secret generic git-credentials \
  --from-literal=GIT_TOKEN=ghp_your_token_here \
  --from-literal=GIT_USER_NAME="Your Name" \
  --from-literal=GIT_USER_EMAIL="your@email.com" \
  -n project-namespace
```

Then reference this secret in the ProjectSettings UI:
1. Navigate to Project Settings
2. Go to Runner Secrets section
3. Add secret name: `git-credentials`

For comprehensive authentication setup, see [GIT_AUTH_SETUP.md](GIT_AUTH_SETUP.md).
```

**Success Criteria**:
- [ ] Secret creation command provided with example values
- [ ] Namespace parameter clearly shown
- [ ] UI configuration steps documented
- [ ] Links to GIT_AUTH_SETUP.md for details

---

### 5. Post-Deployment Steps
**Purpose**: Guide users through initial configuration after deployment
**Requirements**: FR-012
**Content**:
- Create first project via UI
- Configure runner secrets
- Create first agentic session
- Verify session execution

**Code Block Example**:
```markdown
## Post-Deployment Steps

### 1. Create Your First Project

1. Access the vTeam frontend (see Production Deployment section for URL)
2. Click "New Project"
3. Provide project name and description
4. Project namespace will be created automatically

### 2. Configure Runner Secrets

Projects require runner configuration to execute sessions:

```bash
# Create runner secret in project namespace
oc create secret generic runner-config \
  --from-literal=ANTHROPIC_API_KEY=your-key \
  -n project-namespace
```

Then add this secret in Project Settings → Runner Secrets.

### 3. Create Your First Session

1. Navigate to your project
2. Click "New Session"
3. Provide session description (e.g., "Analyze repository structure")
4. Select repository URL (if git auth configured)
5. Click "Start Session"

### 4. Monitor Session Execution

- Session output streams in real-time via WebSocket
- Results are stored in session history
- View logs: `oc logs -n ambient-code runner-pod-name`
```

**Success Criteria**:
- [ ] Project creation workflow documented
- [ ] Runner secret configuration explained
- [ ] First session creation steps provided
- [ ] Verification and monitoring guidance included

---

### 6. Troubleshooting
**Purpose**: Help users resolve common deployment and runtime issues
**Requirements**: FR-013
**Content**:
- Pods not starting (CrashLoopBackOff, ImagePullBackOff)
- Session failures (missing API key, runner errors)
- WebSocket connection issues
- Link to GitHub issues for reporting new problems

**Code Block Example**:
```markdown
## Troubleshooting

### Pods Not Starting

**Symptoms**: Pods in CrashLoopBackOff or ImagePullBackOff state

**Solutions**:
1. Check pod logs:
   ```bash
   oc logs -n ambient-code <pod-name>
   ```

2. Verify secrets exist:
   ```bash
   oc get secrets -n ambient-code
   ```

3. Check image pull access:
   ```bash
   oc describe pod <pod-name> -n ambient-code
   ```

**Common Causes**:
- Missing ANTHROPIC_API_KEY in secrets
- Image registry authentication issues
- Insufficient resources in cluster

---

### Session Failures

**Symptoms**: Sessions created but fail to execute or produce no output

**Solutions**:
1. Verify runner secrets:
   ```bash
   oc get secrets -n project-namespace
   ```

2. Check operator logs:
   ```bash
   oc logs -n ambient-code deployment/operator
   ```

3. Validate AgenticSession CR status:
   ```bash
   oc get agenticsessions -n project-namespace
   oc describe agenticsession <session-name> -n project-namespace
   ```

**Common Causes**:
- Missing runner secret with ANTHROPIC_API_KEY
- Runner pod failures due to RBAC issues
- Invalid session configuration

---

### WebSocket Connection Errors

**Symptoms**: Frontend cannot connect to backend, session streaming fails

**Solutions**:
1. Verify backend service is running:
   ```bash
   oc get svc backend -n ambient-code
   oc get pods -n ambient-code | grep backend
   ```

2. Check route configuration:
   ```bash
   oc get route -n ambient-code
   ```

3. Test backend health endpoint:
   ```bash
   curl $(oc get route backend -n ambient-code -o jsonpath='{.spec.host}')/health
   ```

**Common Causes**:
- Backend pod not running
- Route misconfiguration
- CORS or WebSocket protocol issues

---

### Reporting Issues

If you encounter issues not covered here, please:
1. Check [GitHub Issues](https://github.com/your-org/vTeam/issues)
2. Collect logs: `oc logs -n ambient-code <pod-name>`
3. Report with logs and reproduction steps
```

**Success Criteria**:
- [ ] Top 3 common issues documented (pods, sessions, WebSocket)
- [ ] Each issue has symptoms, solutions, and common causes
- [ ] Diagnostic commands provided for each issue
- [ ] Link to GitHub issues for reporting new problems

---

## Cross-Reference Requirements

**Outgoing Links**:
- `GIT_AUTH_SETUP.md` (git authentication details)
- `../components/README.md` (component-specific development)
- External: Anthropic Console, Red Hat Console, OpenShift docs

**Link Validation**:
- [ ] All internal links use relative paths
- [ ] All external links are valid and accessible
- [ ] Links render correctly in GitHub markdown viewer

---

## Code Block Requirements

**Language Identifiers Required**:
- `bash` for all shell commands
- `yaml` for Kubernetes resource examples

**Standards**:
- [ ] All code blocks specify language for syntax highlighting
- [ ] Bash commands use consistent namespace (ambient-code by default)
- [ ] Environment variables used for sensitive values (no hardcoded secrets)
- [ ] Expected outputs documented where helpful

---

## Consistency Requirements (FR-018, FR-019)

**Namespace References**:
- [ ] Default namespace is `ambient-code` in all examples
- [ ] Project namespace referenced as `project-namespace` when generic
- [ ] Document namespace customization where applicable

**Registry References**:
- [ ] Pre-built images reference `quay.io/ambient_code`
- [ ] No hardcoded registry in user-facing commands

---

## Self-Contained Requirements (FR-022, FR-023)

**Progressive Disclosure**:
- [ ] Quick start at beginning (minimal steps to get running)
- [ ] Detailed configuration later (customization, troubleshooting)
- [ ] No assumption of prior OpenShift/Kubernetes knowledge
- [ ] Links to external resources for learning more

**Completeness**:
- [ ] User can follow guide end-to-end without external context
- [ ] All commands are copy-paste ready
- [ ] Verification steps after each major action

---

## Validation Checklist

Before marking this contract as complete:
- [ ] All 6 required sections present with comprehensive content
- [ ] Prerequisites list all tools with install links
- [ ] Local development workflow is complete (CRC → deploy → verify)
- [ ] Production deployment workflow is complete (login → deploy → access)
- [ ] Git authentication documented with example commands
- [ ] Post-deployment steps cover project creation and first session
- [ ] Troubleshooting covers top 3 common issues with solutions
- [ ] All cross-references use valid paths
- [ ] Code blocks have language identifiers and consistent formatting
- [ ] Namespace and registry references are consistent
- [ ] Functional requirements FR-008 through FR-013 fully addressed

---

**Contract Version**: 1.0
**Generated**: 2025-10-10 by /plan command execution
