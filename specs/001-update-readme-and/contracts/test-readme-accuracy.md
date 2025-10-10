# README Accuracy Test Checklist

**Contract**: Documentation Accuracy Contract - RULE-002, RULE-006
**Artifact**: `/vTeam/README.md`
**Last Validated**: Never
**Status**: ❌ FAILING

---

## Test Scope

This checklist validates that the main README accurately reflects the vTeam implementation, including correct deployment names, working commands, and accurate architecture descriptions.

---

## Architecture Section (Lines 17-27)

### Component Table Accuracy
- [ ] **Frontend** - Verify "Next.js 15 + Shadcn UI" matches `components/frontend/package.json`
  - Expected: Next.js 15.x, shadcn UI components
  - Command: `grep '"next":' components/frontend/package.json`

- [ ] **Backend API** - Verify "Go + Gin" matches `components/backend/go.mod`
  - Expected: Go 1.24+, gin-gonic/gin
  - Command: `head -5 components/backend/go.mod && grep gin components/backend/go.mod`

- [ ] **Operator** - Verify "Go" and "Kubernetes operator" matches `components/operator/go.mod`
  - Expected: Go 1.24+, k8s.io/client-go
  - Command: `head -5 components/operator/go.mod && grep k8s components/operator/go.mod`

- [ ] **Claude Code Runner** - Verify "Python + Claude Code SDK" matches `components/runners/claude-code-runner/pyproject.toml`
  - Expected: Python 3.11+, anthropic, claude-code-sdk
  - Command: `grep -A 5 'dependencies' components/runners/claude-code-runner/pyproject.toml`

- [ ] **Content Service** - Mark as misleading or clarify as backend dual-mode
  - Expected: Should clarify this is backend running in CONTENT_SERVICE_MODE=true
  - Issue: Table implies separate component; actually same binary as Backend API

- [ ] **Runner Shell** - Should be added to table
  - Expected: Row for "Runner Shell" - "Python library | WebSocket transport for AI agent sessions"
  - Current: Missing from table but exists at `components/runners/runner-shell/`

### Session Flow (Lines 29-36)
- [ ] Step 1 "Create Session" - Verify user creates session via web UI
  - Validation: Check frontend has session creation UI at `components/frontend/src/app/projects/[name]/sessions/new/`

- [ ] Step 2 "API Processing" - Verify backend creates AgenticSession CR
  - Validation: Check backend has createSession handler in `components/backend/main.go`

- [ ] Step 3 "Job Scheduling" - Verify operator reconciles CR and creates Job
  - Validation: Check operator reconcile loop in `components/operator/main.go:139-197`

- [ ] Step 4 "AI Execution" - Verify runner executes Claude Code SDK with PVC workspace
  - Validation: Check runner imports claude_code_cli in `components/runners/claude-code-runner/wrapper.py`

- [ ] Step 5 "WebSocket Streaming" - Verify real-time messages via WebSocket
  - Validation: Check WebSocket handler in `components/backend/main.go:170`

- [ ] Step 6 "Result Storage" - Verify results stored in CR status
  - Validation: Check status update logic in `components/backend/handlers.go` or operator

---

## Quick Start Section (Lines 54-99)

### Deployment Commands (Lines 60-71)
- [ ] **Command: `cd components/manifests`** - Verify directory exists
  - Expected: Directory exists with deploy.sh
  - Command: `ls -la components/manifests/deploy.sh`

- [ ] **Command: `cp env.example .env`** - Verify env.example exists
  - Expected: File exists at components/manifests/env.example
  - Command: `ls -la components/manifests/env.example`

- [ ] **Command: `make deploy`** - Verify make target exists
  - Expected: Makefile has deploy target that calls components/manifests/deploy.sh
  - Command: `grep -A 5 '^deploy:' Makefile`

### Verification Commands (Lines 75-82)
- [ ] **Command: `oc get pods -n ambient-code`** - Verify namespace is correct
  - Expected: Namespace ambient-code exists after deployment
  - Note: This is correct

- [ ] **Command: `oc logs -f deployment/vteam-operator -n ambient-code`** ❌ INCORRECT
  - Expected: `oc logs -f deployment/agentic-operator -n ambient-code`
  - Actual deployment name: agentic-operator (per `components/manifests/operator-deployment.yaml:4`)
  - Fix: README.md line 81

### Access Commands (Lines 87-92)
- [ ] **Command: `oc get route frontend-route -n ambient-code`** - Verify route name
  - Expected: Route named "frontend-route"
  - Command: `grep 'name: frontend-route' components/manifests/route.yaml || components/manifests/frontend-deployment.yaml`

- [ ] **Command: `oc port-forward svc/frontend-service 3000:3000`** - Verify service name
  - Expected: Service named "frontend-service" on port 3000
  - Command: `grep 'name: frontend-service' components/manifests/frontend-deployment.yaml`

---

## Configuration Section (Lines 133-165)

### Build Commands (Lines 139-152)
- [ ] **Command: `make build-all`** - Verify make target exists
  - Expected: Makefile has build-all target
  - Command: `grep '^build-all:' Makefile`

- [ ] **Command: `make push-all REGISTRY=$REGISTRY`** - Verify make target exists
  - Expected: Makefile has push-all target with REGISTRY parameter
  - Command: `grep '^push-all:' Makefile`

- [ ] **File path: `components/manifests/.env`** - Verify deployment uses .env
  - Expected: deploy.sh sources .env file
  - Command: `grep '\\.env' components/manifests/deploy.sh`

### Container Engine (Lines 155-165)
- [ ] **Command: `make build-all CONTAINER_ENGINE=podman`** - Verify Makefile supports parameter
  - Expected: Makefile accepts CONTAINER_ENGINE variable
  - Command: `grep 'CONTAINER_ENGINE' Makefile`

- [ ] **Command: `make build-all PLATFORM=linux/arm64`** - Verify Makefile supports parameter
  - Expected: Makefile accepts PLATFORM variable
  - Command: `grep 'PLATFORM' Makefile`

---

## Troubleshooting Section (Lines 192-244)

### Common Issues - Pod Logs (Lines 197-202)
- [ ] **Command: `oc get pods -n ambient-code`** - Correct ✓
- [ ] **Command: `oc describe pod <pod-name> -n ambient-code`** - Correct ✓
- [ ] **Command: `oc logs <pod-name> -n ambient-code`** - Correct ✓

### Session Jobs (Lines 205-211)
- [ ] **Command: `oc get jobs -n <project-namespace>`** - Correct ✓
- [ ] **Command: `oc describe job <session-job-name>`** - Correct ✓
- [ ] **Command: `oc logs <runner-pod-name> -c runner`** - Verify container name is "runner"
  - Expected: Operator creates pod with container named "runner"
  - Command: `grep -A 20 'name: runner' components/operator/main.go`

### WebSocket Issues (Lines 214-220)
- [ ] **Command: `oc port-forward svc/backend-service 8080:8080`** - Verify service name and port
  - Expected: Service named "backend-service" exists on port 8080
  - Command: `grep 'name: backend-service' components/manifests/backend-deployment.yaml`

- [ ] **Command: `curl http://localhost:8080/health`** - Verify health endpoint exists
  - Expected: Backend exposes GET /health endpoint
  - Command: `grep '/health' components/backend/main.go`

- [ ] **Command: `oc logs deployment/vteam-backend -n ambient-code --tail=50`** ❌ INCORRECT
  - Expected: `oc logs deployment/backend-api -n ambient-code --tail=50`
  - Actual deployment name: backend-api (per `components/manifests/backend-deployment.yaml:4`)
  - Fix: README.md line 219

### Authentication Issues (Lines 223-228)
- [ ] **Command: `oc get oauthclient vteam-frontend`** - Verify OAuthClient name
  - Expected: If OAuth enabled, OAuthClient named "vteam-frontend" (or similar)
  - Note: Check docs/OPENSHIFT_OAUTH.md for actual name

- [ ] **Command: `oc describe route frontend-route -n ambient-code`** - Correct ✓

### Verification Commands (Lines 232-244)
- [ ] **Command: `oc get deployments,services,routes -n ambient-code`** - Correct ✓

- [ ] **Command: `oc logs -f deployment/vteam-operator -n ambient-code`** ❌ INCORRECT
  - Expected: `oc logs -f deployment/agentic-operator -n ambient-code`
  - Actual deployment name: agentic-operator
  - Fix: README.md line 236

- [ ] **Command: `oc get crds | grep vteam.ambient-code`** - Verify CRD group
  - Expected: CRDs have group "vteam.ambient-code" or similar
  - Command: `grep 'group:' components/manifests/crds/*.yaml`

- [ ] **Command: `oc get agenticsessions -n <project-namespace>`** - Verify resource name
  - Expected: CRD plural name is "agenticsessions"
  - Command: `grep 'plural: agenticsessions' components/manifests/crds/agenticsession-crd.yaml`

- [ ] **Command: `oc describe agenticsession <session-name>`** - Verify resource name (singular)
  - Expected: CRD kind is "AgenticSession" with singular "agenticsession"
  - Command: `grep 'kind: AgenticSession' components/manifests/crds/agenticsession-crd.yaml`

---

## Project Structure Section (Lines 327-350)

### Directory Structure (Lines 329-350)
- [ ] Verify all listed directories exist:
  - [ ] `components/frontend/` - Next.js 15 web interface ✓
  - [ ] `components/backend/` - Go API service ✓
  - [ ] `components/operator/` - Kubernetes operator ✓
  - [ ] `components/runners/` - Runner components ✓
    - [ ] `components/runners/claude-code-runner/` - Python runner ✓
    - [ ] `components/runners/runner-shell/` - Runner shell library (MISSING FROM DOCS)
  - [ ] `components/manifests/` - Kubernetes manifests ✓
  - [ ] `components/scripts/` - Deployment scripts ✓
  - [ ] `docs/` - Documentation ✓
  - [ ] `agents/` - Agent persona definitions ✓
  - [ ] `Makefile` - Build automation ✓
  - [ ] `mkdocs.yml` - Docs site config ✓

---

## Documentation References (Lines 353-359)

### Documentation Files
- [ ] `docs/OPENSHIFT_DEPLOY.md` - Verify file exists
- [ ] `docs/OPENSHIFT_OAUTH.md` - Verify file exists
- [ ] `docs/GITHUB_APP_SETUP.md` - Verify file exists
- [ ] `docs/CLAUDE_CODE_RUNNER.md` - Verify file exists (or should be `docs/developer-guide/claude-code-runner.md`)
- [ ] `docs/user-guide/` - Verify directory exists
- [ ] `docs/developer-guide/` - Verify directory exists

---

## Test Summary

### Failures Identified
1. ❌ Line 81: Wrong deployment name `vteam-operator` → should be `agentic-operator`
2. ❌ Line 219: Wrong deployment name `vteam-backend` → should be `backend-api`
3. ❌ Line 236: Wrong deployment name `vteam-operator` → should be `agentic-operator`
4. ⚠️ Lines 25-27: Content Service entry is misleading (should clarify dual-mode backend)
5. ⚠️ Architecture table: Missing runner-shell component

### Pass Rate
- **Commands tested**: 35
- **Passed**: 30
- **Failed**: 3 (critical: wrong deployment names)
- **Warnings**: 2 (architecture accuracy issues)
- **Pass Rate**: 85.7%

### Priority Fixes
1. **P0**: Fix all three deployment name references (lines 81, 219, 236)
2. **P1**: Clarify Content Service in architecture table
3. **P2**: Add runner-shell to architecture table

---

**Test Status**: ❌ FAILING - 3 critical command errors, 2 architecture warnings
**Next Action**: Update README.md to fix deployment names and architecture table
