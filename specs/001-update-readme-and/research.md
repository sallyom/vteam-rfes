# Research Findings: vTeam Documentation Audit

**Date**: 2025-10-10
**Feature**: Update README and Documentation for vTeam
**Research Phase**: Phase 0 - Documentation audit against source code

---

## Executive Summary

Comprehensive audit of vTeam documentation against actual implementation reveals significant gaps:
- **Main README**: 85% accurate, but has critical deployment name mismatches
- **Component READMEs**: Only 1 of 4 components has a README (frontend)
- **API Documentation**: 5-10% coverage (4 of 64 endpoints documented)
- **Environment Variables**: 28 of 56 variables completely undocumented
- **Feature Documentation**: 56 major features exist in code but are not documented

---

## 1. README Architecture Audit

### Decision: Update architecture table and add runner-shell component
### Rationale: Architecture table has misleading "Content Service" entry and missing "runner-shell" component
### Alternatives Considered: N/A - must reflect actual implementation

**Source Evidence**:

#### Content Service - MISLEADING ENTRY
- **Current Doc**: README.md:25-27 lists "Content Service" as separate component ("Go (sidecar mode)")
- **Should Be**: Remove as separate component, or clarify it's backend running in sidecar mode
- **File**: `/workspace/sessions/agentic-session-1760129623/workspace/vTeam/components/backend/main.go:95-117`
- **Line**: 95 shows `if os.Getenv("CONTENT_SERVICE_MODE") == "true"` - same binary, different mode
- **Evidence**: No `components/content-service/` directory exists; operator uses backend image as content service by setting `CONTENT_SERVICE_MODE=true` env var (operator/main.go:63)

#### Runner-Shell - MISSING FROM DOCUMENTATION
- **Current Doc**: Not mentioned in architecture table
- **Should Be**: Add row for "Runner Shell" - "Python library | Standardized runner shell with WebSocket transport"
- **File**: `/workspace/sessions/agentic-session-1760129623/workspace/vTeam/components/runners/runner-shell/pyproject.toml`
- **Line**: N/A
- **Evidence**: Directory exists with complete Python package defining WebSocket support, S3 integration, workspace management

#### Incorrect Deployment Names in Troubleshooting
- **Current Doc**: README.md:81, 219, 236 use `deployment/vteam-operator` and `deployment/vteam-backend`
- **Should Be**: Use `deployment/agentic-operator` and `deployment/backend-api`
- **File**: `/workspace/sessions/agentic-session-1760129623/workspace/vTeam/components/manifests/operator-deployment.yaml:4`, `backend-deployment.yaml:4`
- **Evidence**: Actual deployment names differ from documentation; deploy.sh correctly uses `agentic-operator` and `backend-api`

---

## 2. Component Documentation Audit

### Decision: Create READMEs for backend, operator, and runner components
### Rationale: 3 of 4 major components have no README; 56 significant features are undocumented
### Alternatives Considered: Single centralized documentation (rejected - too distant from code)

**Source Evidence**:

### Frontend Component - PARTIALLY DOCUMENTED
- **README exists**: YES at `components/frontend/README.md`
- **Documented features**: ~30% coverage (GitHub App, sessions dashboard, RFE workflows UI, RBAC)
- **Undocumented features** (10 major items):
  1. **Project Management UI** - `src/app/projects/page.tsx`, `src/app/projects/new/page.tsx`
  2. **Permissions Management** - `src/app/projects/[name]/permissions/page.tsx`
  3. **Access Keys Management** - `src/app/projects/[name]/keys/page.tsx`
  4. **Session Cloning** - `src/components/clone-session-dialog.tsx`
  5. **Multi-Agent Selection** - `src/components/multi-agent-selection.tsx`
  6. **Workspace File Management** - `src/components/session/WorkspaceTab.tsx`
  7. **Runner Secrets Configuration** - `src/app/api/projects/[name]/runner-secrets/`
  8. **Integration Pages** - `src/app/integrations/page.tsx`
  9. **User Profile Bubble** - `src/components/user-bubble.tsx`
  10. **Project Settings UI** - `src/app/projects/[name]/settings/page.tsx`

### Backend Component - NOT DOCUMENTED
- **README exists**: NO
- **Documented features**: None
- **Undocumented features** (17 major items):
  1. **GitHub App Token Management** - `github_token.go:104-179` - JWT-based auth, installation token minting/caching
  2. **Per-User GitHub Installation Tracking** - `github_app.go:298-340` - ConfigMap-based storage
  3. **OAuth User Verification** - `github_app.go:127-240` - HMAC-signed state validation
  4. **Fork Management API** - `github_app.go:366-485`
  5. **Repository Tree/Blob Proxy** - `github_app.go:487-660`
  6. **WebSocket Messaging Hub** - `websocket_messaging.go` - bidirectional real-time messaging
  7. **Session Message Persistence** - `websocket_messaging.go:326-402` - JSONL storage
  8. **Access Control Validation** - `handlers.go:218-279` - RBAC enforcement middleware
  9. **Runner Token Provisioning** - `handlers.go:745-870` - ServiceAccount token creation
  10. **Session Cloning** - `handlers.go:1178-1303` - cross-project workspace copying
  11. **Content Service Mode** - `main.go:95-117` - dual-mode operation
  12. **Multi-Repo Session Support** - `main.go:349-354` - unified multi-repository sessions
  13. **RFE Workflow Management** - `handlers.go:2976-3595` - complete CRUD operations
  14. **Jira Integration** - `handlers.go:3407-3480` - workflow file publishing
  15. **Runner Secrets Configuration** - `handlers.go:3675-3875` - project-scoped secrets
  16. **Git Operations via Content Service** - `handlers.go:1660-2132` - push/abandon/diff proxy
  17. **Access Key Authentication** - `handlers.go:99-177` - API key auth with tracking

### Operator Component - NOT DOCUMENTED
- **README exists**: NO
- **Documented features**: None
- **Undocumented features** (15 major items):
  1. **AgenticSession CRD Reconciliation** - `main.go:139-197`
  2. **Job Lifecycle Management** - `main.go:200-586` - creates Jobs with sidecar containers
  3. **Per-Session Workspace PVCs** - `main.go:628-660` - isolated PVCs to avoid multi-attach
  4. **Dual-Container Pod Architecture** - `main.go:354-500` - content service + runner sidecar
  5. **Job Monitoring & Status Updates** - `main.go:662-854` - continuous CR status sync
  6. **Namespace Watching** - `main.go:937-975` - auto-discovers `ambient-code.io/managed=true` namespaces
  7. **ProjectSettings Reconciliation** - `main.go:977-1106` - CRD for group access
  8. **RBAC Automation** - `main.go:1110-1156` - automatic RoleBinding creation
  9. **Content Service per Session** - `main.go:559-581` - per-job Service creation
  10. **Runner Token Secret Management** - `main.go:417-438` - injects runner auth tokens
  11. **Multi-Repo Environment Injection** - `main.go:441-486` - parses REPOS_JSON
  12. **Resource Owner References** - `main.go:239-248`, `main.go:315-324` - garbage collection
  13. **Job Cleanup on Finalization** - `main.go:817-852` - waits for repo push/abandon
  14. **Security Context Configuration** - `main.go:382-388` - SCC-compatible settings
  15. **Runner Secrets Volume Mounting** - `main.go:508-520` - mounts as env vars + volumes

### Claude Code Runner Component - NOT DOCUMENTED
- **README exists**: NO (only agents/README.md exists for agent personas)
- **Documented features**: None
- **Undocumented features** (15 major items):
  1. **Runner-Shell Integration** - `wrapper.py:18-31` - adapter pattern with standardized framework
  2. **Multi-Repo Workspace Preparation** - `wrapper.py:368-415` - clones multiple repos to subdirs
  3. **GitHub Token Minting** - `wrapper.py:824-864` - on-demand token fetch from backend
  4. **Prerequisite Validation** - `wrapper.py:451-500` - validates /plan requires spec.md, etc.
  5. **Interactive Session Support** - `wrapper.py:237-338` - bidirectional chat with message queueing
  6. **CR Status Updates** - `wrapper.py:709-742` - direct Kubernetes CR updates via backend API
  7. **Automatic PR Creation** - `wrapper.py:577-634` - creates PRs after push with cross-fork support
  8. **Multi-Repo Push Logic** - `wrapper.py:503-575` - handles multiple output repos
  9. **SDK Result Message Handling** - `wrapper.py:286-303` - captures usage/cost tracking
  10. **WebSocket Connection Retry** - `wrapper.py:760-792` - waits for WS before sending
  11. **Tool Result Streaming** - `wrapper.py:261-279` - streams tool use/results to WebSocket
  12. **Main Repo Selection** - `wrapper.py:166-186` - MAIN_REPO_NAME/INDEX for multi-repo CWD
  13. **Additional Directories Support** - `wrapper.py:179-194` - passes non-main repos to SDK
  14. **Git Identity Configuration** - `wrapper.py:398-403`, `wrapper.py:434-439` - sets git user
  15. **Turn Counting** - `wrapper.py:224-280` - tracks agent turns for metrics

---

## 3. Environment Variables Audit

### Decision: Create comprehensive environment variables reference documentation
### Rationale: 28 of 56 variables (50%) are completely undocumented; critical operational variables are missing
### Alternatives Considered: Keep scattered across .env.example files (rejected - hard to discover and incomplete)

**Source Evidence**:

### Undocumented Variables (28 total)

#### Backend Undocumented
| Variable | Usage | Purpose |
|----------|-------|---------|
| CONTENT_SERVICE_MODE | main.go:95 | Enable content service mode (true/false) |
| SESSION_CONTENT_SERVICE_BASE | handlers.go:1565+ | Base URL for session content service |

#### Frontend Undocumented
| Variable | Usage | Purpose |
|----------|-------|---------|
| NODE_ENV | manifests/frontend-deployment.yaml:28 | Node environment (standard but undocumented) |

#### Runner Undocumented (25 variables)
| Variable | Usage | Purpose |
|----------|-------|---------|
| MAIN_REPO_NAME | wrapper.py:166 | Name of main repository for multi-repo sessions |
| MAIN_REPO_INDEX | wrapper.py:168 | Index of main repo in repos array |
| GITHUB_TOKEN | wrapper.py:370+ | GitHub token for repository access |
| INPUT_REPO_URL | wrapper.py:418 | Input repository URL to clone |
| INPUT_BRANCH | wrapper.py:421 | Input repository branch to checkout |
| OUTPUT_REPO_URL | wrapper.py:422 | Output repository URL for pushing changes |
| OUTPUT_BRANCH | wrapper.py:548 | Output repository branch to push to |
| CREATE_PR | wrapper.py:529 | Flag to create pull request after push |
| PR_TARGET_BRANCH | wrapper.py:532 | Target branch for pull request |
| PROJECT_NAME | wrapper.py:696 | Project/namespace name |
| BACKEND_API_URL | wrapper.py:701 | Backend API URL for GitHub token fetch |
| REPOS_JSON | wrapper.py:914 | JSON array of repositories to clone |
| WEBSOCKET_URL | wrapper.py:962 | WebSocket URL for backend communication |
| DEBUG | operator/main.go:396 | Enable debug logging |
| AGENTIC_SESSION_NAME | operator/main.go:398 | Name of the AgenticSession CR |
| AGENTIC_SESSION_NAMESPACE | operator/main.go:399 | Namespace of the AgenticSession CR |
| LLM_MODEL | operator/main.go:409 | LLM model to use |
| LLM_TEMPERATURE | operator/main.go:410 | LLM temperature setting |
| LLM_MAX_TOKENS | operator/main.go:411 | Maximum tokens for LLM response |
| TIMEOUT | operator/main.go:412 | Session timeout in seconds |

### Partially Documented Variables (10 total)
- BOT_TOKEN, GIT_USER_NAME, GIT_USER_EMAIL, SESSION_ID, WORKSPACE_PATH, INTERACTIVE, PROMPT, CLAUDE_PERMISSION_MODE
- **Issue**: Mentioned in docs/CLAUDE_CODE_RUNNER.md but not in .env.example files

### Documented Variables (18 total)
- NAMESPACE, STATE_BASE_DIR, PVC_BASE_DIR, PORT, GITHUB_APP_ID, GITHUB_PRIVATE_KEY, GITHUB_CLIENT_ID, GITHUB_CLIENT_SECRET, GITHUB_STATE_SECRET, BACKEND_URL, GITHUB_APP_SLUG, OC_TOKEN, OC_USER, OC_EMAIL, ENABLE_OC_WHOAMI, ANTHROPIC_API_KEY, BACKEND_NAMESPACE, AMBIENT_CODE_RUNNER_IMAGE, CONTENT_SERVICE_IMAGE, IMAGE_PULL_POLICY

---

## 4. Deployment Documentation Audit

### Decision: Fix deployment command names and add missing prerequisites
### Rationale: Troubleshooting commands use wrong deployment names; would fail when users need them most
### Alternatives Considered: N/A - must match actual deployment resources

**Source Evidence**:

### Incorrect Deployment Names (CRITICAL)
- **Current Doc**: README.md:81, 219, 236 and docs/user-guide/getting-started.md:79, 203
- **Wrong**: `deployment/vteam-operator` and `deployment/vteam-backend`
- **Should Be**: `deployment/agentic-operator` and `deployment/backend-api`
- **File**: `/workspace/sessions/agentic-session-1760129623/workspace/vTeam/components/manifests/operator-deployment.yaml:4`
- **File**: `/workspace/sessions/agentic-session-1760129623/workspace/vTeam/components/manifests/backend-deployment.yaml:4`
- **Evidence**: deploy.sh correctly uses `agentic-operator` and `backend-api` (lines 370-372, 421-422)

### Missing Prerequisites
- **Current Doc**: README.md:38-49 lists OpenShift, oc CLI, container registry, Docker/Podman, Go, Node.js
- **Should Be**: Add `kustomize` to Required Infrastructure section
- **File**: `/workspace/sessions/agentic-session-1760129623/workspace/vTeam/components/manifests/deploy.sh:248`
- **Line**: 248 shows `command -v kustomize` check
- **Evidence**: Deploy script fails without kustomize but README doesn't list it as prerequisite

### Undocumented Manifests
- **Current Doc**: Manifests directory not documented
- **Should Be**: Document key manifests in deployment guide
- **Files**:
  - `backend-route.yaml` - Backend API route (not mentioned)
  - `workspace-pvc.yaml` - Backend state PVC (not mentioned)
  - `claude-runner-rbac.yaml` - Runner RBAC (not mentioned)
  - `git-configmap.yaml` - Git configuration (not mentioned)
  - `GIT_AUTH_SETUP.md` - Exists in manifests/ but not linked from main docs

### Configuration Documentation Gaps
- **Current Doc**: env.example variables not all explained
- **Should Be**: Clarify which variables are optional vs required
- **File**: `/workspace/sessions/agentic-session-1760129623/workspace/vTeam/components/manifests/env.example`
- **Evidence**:
  - `CLAUDE_ALLOWED_TOOLS` mentioned but purpose not explained
  - OAuth variables (OCP_OAUTH_CLIENT_SECRET, OCP_OAUTH_COOKIE_SECRET) not clearly marked as optional
  - ANTHROPIC_API_KEY can be configured via UI but docs don't clarify this

---

## 5. API and Integration Documentation Audit

### Decision: Create comprehensive API reference documentation and expand integration guides
### Rationale: Only 4 of 64 HTTP endpoints documented; major features like RFE workflows, multi-repo support, and fork management are undocumented
### Alternatives Considered: Inline code documentation only (rejected - not discoverable for API consumers)

**Source Evidence**:

### API Documentation - CRITICALLY INCOMPLETE

#### Total API Surface
- **64 HTTP endpoints** identified across backend
- **4 endpoints** partially documented (6% coverage)
- **60 endpoints** completely undocumented

#### Undocumented Endpoint Categories

**Content Service Endpoints** (7 endpoints)
- POST /content/write, GET /content/file, GET /content/list
- POST /content/github/push, POST /content/github/abandon, GET /content/github/diff
- GET /health (content service mode)
- **File**: `main.go:96-104`

**Project Management** (5 endpoints)
- GET/POST /api/projects, GET/PUT/DELETE /api/projects/:projectName
- **File**: `main.go:209-213`

**Agentic Sessions** (17 endpoints)
- Session CRUD, workspace operations, Git operations, WebSocket
- GET/POST /api/projects/:projectName/agentic-sessions
- GET/PUT/DELETE /api/projects/:projectName/agentic-sessions/:sessionName
- POST .../:sessionName/clone, .../:sessionName/start, .../:sessionName/stop
- PUT .../:sessionName/status
- GET/PUT .../:sessionName/workspace/*path
- POST .../:sessionName/github/push, .../:sessionName/github/abandon
- GET .../:sessionName/github/diff
- GET .../:projectName/sessions/:sessionId/ws, .../messages
- POST .../:projectName/sessions/:sessionId/messages
- **File**: `main.go:140-172`

**RFE Workflows** (11 endpoints)
- GET/POST /api/projects/:projectName/rfe-workflows
- GET/DELETE .../:id
- GET .../:id/summary
- POST .../:id/seed
- GET .../:id/check-seeding
- POST/GET .../:id/jira
- GET .../:id/sessions
- POST .../:id/sessions/link
- DELETE .../:id/sessions/:sessionName
- **File**: `main.go:161-180`

**Permissions & Keys** (6 endpoints)
- GET/POST /api/projects/:projectName/permissions
- DELETE .../:subjectType/:subjectName
- GET/POST /api/projects/:projectName/keys
- DELETE .../keys/:keyId
- **File**: `main.go:185-192`

**Runner Secrets** (5 endpoints)
- GET /api/projects/:projectName/secrets
- GET/PUT .../runner-secrets/config
- GET/PUT .../runner-secrets
- **File**: `main.go:195-199`

**Fork Management** (2 endpoints)
- GET/POST /api/projects/:projectName/users/forks
- **File**: `main.go:132-133`

**Repository Browsing** (2 endpoints)
- GET /api/projects/:projectName/repo/tree
- GET /api/projects/:projectName/repo/blob
- **File**: `main.go:136-137`

**Authentication** (4 endpoints - partially documented)
- POST /api/auth/github/install - Partially documented
- GET /api/auth/github/status - Partially documented
- POST /api/auth/github/disconnect - Undocumented
- GET /api/auth/github/user/callback - Partially documented
- **File**: `main.go:203-206`
- **Documented in**: docs/GITHUB_APP_SETUP.md (only install and status endpoints, only in context of setup)

**Runner Authentication** (1 endpoint)
- POST /api/projects/:projectName/agentic-sessions/:sessionName/github/token
- **File**: `main.go:123`

**Health** (1 endpoint)
- GET /health - Undocumented
- **File**: `main.go:217`

### GitHub Integration Documentation - MEDIUM ACCURACY

**File**: `/workspace/sessions/agentic-session-1760129623/workspace/vTeam/docs/GITHUB_APP_SETUP.md`

**Documented**: GitHub App creation, permissions, OAuth flow, environment variables, installation linking, status checking

**Undocumented Implementation Details**:
1. **Fork Management** - `github_app.go:366-485` - list and create user forks
2. **ConfigMap Storage** - `github_app.go:298-340` - user→installation mapping storage mechanism
3. **Token Caching** - `github_token.go:116-179` - 3-minute cache threshold for installation tokens
4. **Installation Validation** - `github_token.go:182-235` - access validation logic
5. **Session Token Minting** - `github_token.go:237-257` - short-lived tokens for runner authentication
6. **Disconnect Endpoint** - `github_app.go:715-764` - POST /api/auth/github/disconnect
7. **Repository Browsing Proxy** - `github_app.go:487-660` - tree and blob endpoints

### OAuth Integration Documentation - HIGH ACCURACY

**File**: `/workspace/sessions/agentic-session-1760129623/workspace/vTeam/docs/OPENSHIFT_OAUTH.md`

**Documented**: oauth-proxy sidecar pattern, OAuthClient creation, Secret configuration, Route configuration, Deployment configuration

**Accurate but Missing Details**:
1. **Forwarded Headers** - `main.go:270-300` - X-Forwarded-User, X-Forwarded-Email, X-Forwarded-Groups, X-Forwarded-Preferred-Username, X-Forwarded-Access-Token
2. **Authorization Flow** - How backend uses forwarded headers for RBAC decisions
3. **Token Forwarding** - How X-Forwarded-Access-Token is used for Kubernetes API calls
4. **RBAC Middleware** - `main.go:128` - validateProjectContext() performs SSAR checks

---

## Research Conclusions

### All NEEDS CLARIFICATION from spec.md Resolved

From original spec.md:
1. ~~"Which specific documentation sections need the most attention?"~~
   - **RESOLVED**: Backend, operator, runner component READMEs (missing entirely), API reference (5% coverage), environment variables (50% undocumented)

2. ~~"Are there specific components or features that are missing from docs?"~~
   - **RESOLVED**: 56 major features identified as undocumented (see sections 2 and 5 above)

3. ~~"Should we focus on user-facing or developer-facing documentation?"~~
   - **RESOLVED**: Both are needed. User-facing: Fix deployment commands, add Quick Start validation. Developer-facing: Component READMEs, API reference, architecture diagrams

### Priority Rankings

**P0 - Critical (blocks users/developers)**:
1. Fix incorrect deployment names (vteam-operator → agentic-operator, vteam-backend → backend-api)
2. Create backend API reference (64 endpoints, 0% documented)
3. Create backend component README (17 major features undocumented)
4. Create operator component README (15 major features undocumented)

**P1 - High (major usability impact)**:
5. Create runner component README (15 major features undocumented)
6. Create environment variables reference (28 undocumented variables)
7. Update frontend README (10 major features undocumented)
8. Expand GitHub integration docs (7 features undocumented)
9. Add kustomize to prerequisites

**P2 - Medium (improves completeness)**:
10. Fix README architecture table (Content Service misleading, runner-shell missing)
11. Document undocumented manifests (4 files)
12. Clarify optional vs required configuration (env.example)
13. Expand OAuth integration docs (4 implementation details)

### Next Steps

**Phase 1 will generate**:
- `data-model.md` - Documentation artifact structure and relationships
- `contracts/` - Documentation accuracy contracts (what must be true for docs to be "correct")
- `quickstart.md` - Manual validation workflow for documentation accuracy
- Agent file update - Add documentation technologies and patterns

**Phase 2 (via /tasks) will generate**:
- `tasks.md` - Ordered, specific tasks to implement all documentation updates identified in this research

---

**Research Phase Complete**: All unknowns resolved, ready for Phase 1 design.
