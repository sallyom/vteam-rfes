# Documentation Review Quickstart

**Purpose**: Manual workflow for validating that vTeam documentation accurately reflects the implementation
**Target Audience**: Documentation reviewers, maintainers, contributors
**Estimated Time**: 2-3 hours for complete validation
**Last Updated**: 2025-10-10

---

## Prerequisites

Before starting this validation workflow, ensure you have:

- ✅ Cloned vTeam repository to local machine
- ✅ Cloned vteam-rfes repository to local machine
- ✅ Installed tools: `make`, `oc` CLI, `grep`, `diff`, `jq` (optional)
- ✅ Read access to both repositories
- ✅ Text editor for taking notes

**Repository Paths** (adjust these to your local paths):
```bash
export VTEAM_REPO="/path/to/vTeam"
export RFE_REPO="/path/to/vteam-rfes"
export AUDIT_RESULTS="$RFE_REPO/specs/001-update-readme-and/audit-results.md"
```

---

## Validation Workflow

This quickstart follows a systematic approach to validate documentation accuracy across all vTeam components and guides.

---

### Step 1: README Architecture Validation

**Objective**: Ensure the main README architecture table matches actual component structure

**Procedure**:
1. Open `$VTEAM_REPO/README.md` and locate the architecture table (around lines 20-27)
2. List all components in the repository:
   ```bash
   cd $VTEAM_REPO
   ls -1 components/
   ```
3. For each component in the README table, verify it exists and check technology:
   ```bash
   # Frontend
   cat components/frontend/package.json | grep '"next"'

   # Backend
   head -3 components/backend/go.mod
   grep "gin" components/backend/go.mod

   # Operator
   head -3 components/operator/go.mod
   grep "k8s.io" components/operator/go.mod

   # Claude Code Runner
   grep -A 10 "dependencies" components/runners/claude-code-runner/pyproject.toml

   # Runner Shell (CHECK IF DOCUMENTED)
   ls -la components/runners/runner-shell/
   ```

4. **Compare outputs to README table**:
   - All components listed? ✅ / ❌
   - Technologies match? ✅ / ❌
   - Missing components (like runner-shell)? List:
   - Misleading entries (like "Content Service")? List:

5. **Record findings** in `$AUDIT_RESULTS`:
   ```markdown
   ## Architecture Table Audit
   - Accurate: [YES/NO]
   - Missing components: [list]
   - Misleading entries: [list]
   - Recommended changes: [list]
   ```

---

### Step 2: Quick Start Command Validation

**Objective**: Verify that all commands in the Quick Start section work correctly

**Procedure**:
1. Open `$VTEAM_REPO/README.md` Quick Start section (lines 54-99)
2. Execute each command exactly as written, noting successes and failures:

   ```bash
   cd $VTEAM_REPO

   # Command 1: Check manifests directory
   cd components/manifests && ls -la deploy.sh
   # Result: ✅ / ❌

   # Command 2: Check env.example
   ls -la env.example
   # Result: ✅ / ❌

   # Command 3: Check make deploy target (don't run, just check existence)
   cd $VTEAM_REPO
   grep "^deploy:" Makefile
   # Result: ✅ / ❌

   # Command 4: Verify namespace (conceptual check)
   # Expected: ambient-code namespace mentioned in manifests
   grep -r "namespace: ambient-code" components/manifests/
   # Result: ✅ / ❌

   # Command 5: Check deployment names (CRITICAL)
   grep "name: vteam-operator" components/manifests/operator-deployment.yaml
   # Expected: NOT FOUND (actual name is agentic-operator)
   # Result: ✅ (not found) / ❌ (found - means docs are wrong)

   grep "name: agentic-operator" components/manifests/operator-deployment.yaml
   # Expected: FOUND
   # Result: ✅ / ❌

   grep "name: vteam-backend" components/manifests/backend-deployment.yaml
   # Expected: NOT FOUND (actual name is backend-api)
   # Result: ✅ (not found) / ❌ (found - means docs are wrong)

   grep "name: backend-api" components/manifests/backend-deployment.yaml
   # Expected: FOUND
   # Result: ✅ / ❌

   # Command 6: Check route name
   grep "name: frontend-route" components/manifests/ -r
   # Result: ✅ / ❌

   # Command 7: Check service name
   grep "name: frontend-service" components/manifests/ -r
   # Result: ✅ / ❌
   ```

3. **Record findings**:
   ```markdown
   ## Quick Start Validation
   - All commands work: [YES/NO]
   - Failed commands: [list with line numbers]
   - Incorrect resource names: [list]
   - Missing files: [list]
   ```

---

### Step 3: Component README Validation

**Objective**: Identify which components have READMEs and what features are undocumented

**Procedure**:
1. For each component, check if README exists and evaluate completeness:

   ```bash
   cd $VTEAM_REPO

   # Frontend README
   test -f components/frontend/README.md && echo "EXISTS" || echo "MISSING"
   if [ -f components/frontend/README.md ]; then
     echo "Documented features:"
     grep "^##" components/frontend/README.md
     echo "Source features to check:"
     ls components/frontend/src/app/projects/
     ls components/frontend/src/components/ | head -10
   fi

   # Backend README
   test -f components/backend/README.md && echo "EXISTS" || echo "MISSING"
   if [ -f components/backend/README.md ]; then
     # Evaluate completeness
     echo "Lines: $(wc -l components/backend/README.md)"
   else
     echo "Major files in backend:"
     ls components/backend/*.go
   fi

   # Operator README
   test -f components/operator/README.md && echo "EXISTS" || echo "MISSING"
   if [ -f components/operator/README.md ]; then
     echo "Lines: $(wc -l components/operator/README.md)"
   else
     echo "Major operator features to document:"
     grep "func.*Reconcile" components/operator/main.go
   fi

   # Runner README
   test -f components/runners/claude-code-runner/README.md && echo "EXISTS" || echo "MISSING"
   if [ -f components/runners/claude-code-runner/README.md ]; then
     echo "Lines: $(wc -l components/runners/claude-code-runner/README.md)"
   else
     echo "Major runner features to document:"
     grep "^def " components/runners/claude-code-runner/wrapper.py | head -10
   fi
   ```

2. For frontend (which has a README), identify undocumented features:
   ```bash
   cd $VTEAM_REPO

   # Check if these UI features are documented in frontend README:
   echo "Project Management UI:"
   grep -i "project.*management" components/frontend/README.md || echo "NOT DOCUMENTED"

   echo "Permissions UI:"
   grep -i "permission" components/frontend/README.md || echo "NOT DOCUMENTED"

   echo "Access Keys:"
   grep -i "access.*key\|api.*key" components/frontend/README.md || echo "NOT DOCUMENTED"

   echo "Session Cloning:"
   grep -i "clone" components/frontend/README.md || echo "NOT DOCUMENTED"
   ```

3. **Record findings**:
   ```markdown
   ## Component README Audit

   ### Frontend
   - README exists: [YES/NO]
   - Completeness: [HIGH/MEDIUM/LOW]
   - Undocumented features: [list from src/ exploration]

   ### Backend
   - README exists: [YES/NO]
   - Missing: [YES if no README]
   - Major features to document: [list from .go files]

   ### Operator
   - README exists: [YES/NO]
   - Missing: [YES if no README]
   - Major features to document: [list from main.go]

   ### Runner
   - README exists: [YES/NO]
   - Missing: [YES if no README]
   - Major features to document: [list from wrapper.py]
   ```

---

### Step 4: Configuration Documentation Validation

**Objective**: Ensure all environment variables used in code are documented

**Procedure**:
1. Extract environment variables from source code:

   ```bash
   cd $VTEAM_REPO

   # Backend variables
   echo "=== Backend Environment Variables ==="
   grep -rn "os\.Getenv" components/backend/ | cut -d: -f1,2,3 | sort -u

   # Frontend variables
   echo "=== Frontend Environment Variables ==="
   grep -rn "process\.env\." components/frontend/src/ | cut -d: -f1,2,3 | sort -u

   # Operator variables
   echo "=== Operator Environment Variables ==="
   grep -rn "os\.Getenv" components/operator/ | cut -d: -f1,2,3 | sort -u

   # Runner variables (injected by operator)
   echo "=== Runner Environment Variables ==="
   grep -A 2 "name: [A-Z_]*" components/operator/main.go | grep -E "name:|value:|valueFrom:"
   ```

2. Check if each variable is documented:
   ```bash
   # Check .env.example files
   cat components/manifests/env.example
   test -f components/backend/.env.example && cat components/backend/.env.example
   test -f components/frontend/.env.example && cat components/frontend/.env.example

   # Check if ENVIRONMENT_VARIABLES.md exists
   test -f docs/ENVIRONMENT_VARIABLES.md && echo "EXISTS" || echo "MISSING"
   ```

3. Manually compare extracted variables to documentation:
   - For each variable found in code, check if it appears in .env.example or docs
   - Mark as: DOCUMENTED, PARTIALLY_DOCUMENTED, or UNDOCUMENTED

4. **Record findings**:
   ```markdown
   ## Environment Variables Audit
   - Total variables found: [count]
   - Documented in .env.example: [count]
   - Documented in docs: [count]
   - Undocumented: [count]
   - List of undocumented variables: [list with file:line references]
   ```

---

### Step 5: API Documentation Validation

**Objective**: Identify all HTTP endpoints and check if they're documented

**Procedure**:
1. Extract all HTTP endpoints from backend code:

   ```bash
   cd $VTEAM_REPO

   # Extract all router definitions
   echo "=== Backend API Endpoints ==="
   grep -n "router\.\(GET\|POST\|PUT\|DELETE\|PATCH\)" components/backend/main.go | \
     sed 's/.*router\.\([A-Z]*\)("\([^"]*\)".*/\1 \2/' | \
     sort

   # Count total endpoints
   grep -c "router\.\(GET\|POST\|PUT\|DELETE\|PATCH\)" components/backend/main.go
   ```

2. Check for API documentation:
   ```bash
   # Check if API reference exists
   test -f docs/API_REFERENCE.md && echo "EXISTS" || echo "MISSING"
   test -f docs/api-reference.md && echo "EXISTS" || echo "MISSING"
   test -f docs/developer-guide/api-reference.md && echo "EXISTS" || echo "MISSING"

   # Check OpenAPI/Swagger spec
   find . -name "openapi.yaml" -o -name "swagger.yaml" -o -name "api-spec.yaml"
   ```

3. For existing API docs (if any), spot-check coverage:
   ```bash
   # Example: Check if session endpoints are documented
   grep -i "agentic-session" docs/*.md docs/**/*.md 2>/dev/null | wc -l

   # Check if RFE workflow endpoints are documented
   grep -i "rfe-workflow" docs/*.md docs/**/*.md 2>/dev/null | wc -l
   ```

4. **Record findings**:
   ```markdown
   ## API Documentation Audit
   - Total endpoints: [count from step 1]
   - API reference exists: [YES/NO at path]
   - Endpoints documented: [count or "none"]
   - Documentation coverage: [percentage or "0%"]
   - Critical missing endpoints: [list major endpoint categories]
   ```

---

### Step 6: Deployment Documentation Validation

**Objective**: Verify deployment guide accuracy against actual manifests

**Procedure**:
1. List all manifests:
   ```bash
   cd $VTEAM_REPO
   ls -1 components/manifests/*.yaml
   ls -1 components/manifests/crds/*.yaml
   ls -1 components/manifests/rbac/*.yaml
   ```

2. Check which manifests are mentioned in documentation:
   ```bash
   # Check README
   grep "\.yaml" README.md

   # Check deployment guide
   test -f docs/OPENSHIFT_DEPLOY.md && grep "\.yaml" docs/OPENSHIFT_DEPLOY.md
   ```

3. Verify deploy.sh prerequisites:
   ```bash
   # Extract tools checked by deploy.sh
   grep "command -v" components/manifests/deploy.sh

   # Compare to README Prerequisites
   grep -A 10 "Prerequisites" README.md
   ```

4. **Record findings**:
   ```markdown
   ## Deployment Documentation Audit
   - Total manifest files: [count]
   - Manifests mentioned in docs: [count]
   - Undocumented manifests: [list]
   - Missing prerequisites: [list tools checked by deploy.sh but not in README]
   - Incorrect commands: [list from Step 2]
   ```

---

### Step 7: Integration Documentation Validation

**Objective**: Verify GitHub App and OAuth documentation matches implementation

**Procedure**:
1. Review GitHub integration:
   ```bash
   cd $VTEAM_REPO

   # Check documented features
   cat docs/GITHUB_APP_SETUP.md | grep "^##"

   # Check implemented features
   echo "GitHub App implementation files:"
   ls components/backend/github*.go

   # Check for undocumented features
   grep "^func " components/backend/github_app.go | head -20
   ```

2. Review OAuth integration:
   ```bash
   # Check documented OAuth setup
   cat docs/OPENSHIFT_OAUTH.md | grep "^##"

   # Check forwarded headers in code
   grep "X-Forwarded" components/backend/main.go
   ```

3. **Record findings**:
   ```markdown
   ## Integration Documentation Audit

   ### GitHub App
   - Documentation location: [path]
   - Documented features: [list]
   - Undocumented features: [list from github_app.go exploration]
   - Accuracy: [HIGH/MEDIUM/LOW]

   ### OAuth
   - Documentation location: [path]
   - Documented features: [list]
   - Missing details: [list]
   - Accuracy: [HIGH/MEDIUM/LOW]
   ```

---

## Expected Outcome

After completing all validation steps, you should have:

1. **Audit Results Document** (`$AUDIT_RESULTS`) with:
   - Architecture accuracy assessment
   - List of incorrect commands with corrections
   - Missing component READMEs identified
   - Undocumented environment variables enumerated
   - API documentation coverage measured
   - Deployment guide gaps identified
   - Integration documentation assessed

2. **Summary Metrics**:
   ```markdown
   ## Validation Summary
   - README accuracy: [percentage]
   - Component README coverage: [X of 4 exist]
   - Environment variable documentation: [X of Y documented]
   - API documentation coverage: [X of Y endpoints documented]
   - Deployment guide accuracy: [HIGH/MEDIUM/LOW]
   - Integration guide accuracy: [HIGH/MEDIUM/LOW]
   ```

3. **Prioritized Fix List**:
   - P0 (Critical): [issues that block users]
   - P1 (High): [major usability impacts]
   - P2 (Medium): [improves completeness]
   - P3 (Low): [nice to have]

---

## Next Steps

After completing this quickstart validation:

1. **Review findings** with documentation maintainers
2. **Create GitHub issues** for each identified gap (or use existing RFE)
3. **Generate tasks.md** using `/tasks` command with validated findings
4. **Execute documentation updates** following the ordered task list
5. **Re-run this quickstart** to verify fixes

---

## Maintenance

This quickstart should be re-run:
- ✅ After major feature additions to any component
- ✅ Before major releases
- ✅ Quarterly as part of documentation hygiene
- ✅ When onboarding new documentation maintainers

**Last validation**: Never (this is the initial validation)
**Next scheduled validation**: After documentation updates complete
