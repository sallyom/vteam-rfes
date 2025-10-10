# Quickstart: vTeam Documentation Update Implementation

**Feature**: Update README and Other Docs for vTeam
**Date**: 2025-10-10
**Target Repository**: `/workspace/.../vTeam/`

---

## Overview

This quickstart provides step-by-step instructions for implementing the vTeam documentation updates. Since this is a documentation-only feature, the workflow focuses on creating and updating markdown files based on contracts and research findings.

---

## Prerequisites

- [x] Feature specification complete (spec.md)
- [x] Research complete (research.md)
- [x] Data model defined (data-model.md)
- [x] Documentation contracts created (contracts/*.md)
- [ ] Access to vTeam source repository at `/workspace/.../vTeam/`
- [ ] Understanding of vTeam architecture and deployment mechanisms

---

## Implementation Workflow

### Phase 1: Preparation

1. **Navigate to vTeam repository**:
   ```bash
   cd /workspace/sessions/agentic-session-1760134354/workspace/vTeam
   ```

2. **Create feature branch**:
   ```bash
   git checkout -b docs-update-001
   ```

3. **Verify current documentation state**:
   ```bash
   ls -la README.md components/README.md docs/
   ```

---

### Phase 2: Update README.md

**Contract Reference**: `contracts/README-contract.md`
**Requirements**: FR-001, FR-002, FR-003, FR-004

**Steps**:

1. **Read existing README.md**:
   ```bash
   cat README.md
   ```

2. **Update sections per contract**:
   - Overview: Platform description
   - Architecture: Component table with Next.js 15, Go 1.24, etc.
   - Quick Start: Reference to deploy.sh script
   - Git Authentication: Link to GIT_AUTH_SETUP.md

3. **Validate**:
   - [ ] Component versions match package.json and go.mod
   - [ ] Deploy script path is correct (components/manifests/deploy.sh)
   - [ ] Cross-references use relative paths
   - [ ] Code blocks have language identifiers

---

### Phase 3: Update components/README.md

**Contract Reference**: Derived from FR-005, FR-006, FR-007
**Requirements**: FR-005, FR-006, FR-007

**Steps**:

1. **Read existing components/README.md**:
   ```bash
   cat components/README.md
   ```

2. **Update sections**:
   - Directory Structure: List backend/, frontend/, operator/, runners/, manifests/, scripts/
   - Architecture Flow: 7-step agentic session workflow
   - Quick Start: Production deploy, local dev, custom images

3. **Validate**:
   - [ ] Directory structure reflects actual layout
   - [ ] Architecture flow is accurate and complete
   - [ ] Quick start commands are correct and testable

---

### Phase 4: Create docs/getting-started.md

**Contract Reference**: `contracts/getting-started-contract.md`
**Requirements**: FR-008, FR-009, FR-010, FR-011, FR-012, FR-013

**Steps**:

1. **Create docs directory if needed**:
   ```bash
   mkdir -p docs
   ```

2. **Create getting-started.md** with 6 sections:
   - Prerequisites (tools, API keys)
   - Local Development Setup (CRC installation, configuration, deployment)
   - Production Deployment (login, configure, deploy, verify, access)
   - Git Authentication Configuration (secret creation, UI setup)
   - Post-Deployment Steps (project creation, runner secrets, first session)
   - Troubleshooting (pods, sessions, WebSocket issues)

3. **Validate**:
   - [ ] All 6 sections present and complete
   - [ ] Commands are copy-paste ready
   - [ ] Verification steps after each major action
   - [ ] Troubleshooting covers top 3 issues
   - [ ] Links to GIT_AUTH_SETUP.md are correct

---

### Phase 5: Update docs/OPENSHIFT_DEPLOY.md

**Contract Reference**: Derived from FR-014, FR-015, FR-016, FR-017
**Requirements**: FR-014, FR-015, FR-016, FR-017

**Steps**:

1. **Check if file exists**:
   ```bash
   ls -la docs/OPENSHIFT_DEPLOY.md
   ```

2. **Update or create with sections**:
   - Prerequisites (oc, kustomize, registry access)
   - Quick Deploy (4-step process)
   - Deployment Script Options (standard, custom namespace, secrets-only, uninstall)
   - Git Authentication (emphasize separate secret creation, link to GIT_AUTH_SETUP.md)
   - Post-Deployment Setup (runner secrets, projects, git config)
   - Cleanup (uninstall commands)

3. **Validate**:
   - [ ] Prerequisites explicitly listed
   - [ ] Deploy script options documented with examples
   - [ ] Git authentication section links to GIT_AUTH_SETUP.md
   - [ ] Post-deployment workflow is complete

---

### Phase 6: Cross-Reference Validation

**Requirements**: FR-020

**Steps**:

1. **Extract all links from documentation**:
   ```bash
   grep -r '\[.*\](.*\.md)' README.md components/README.md docs/*.md
   ```

2. **Verify each link**:
   - [ ] README.md → docs/getting-started.md
   - [ ] README.md → docs/OPENSHIFT_DEPLOY.md
   - [ ] README.md → docs/GIT_AUTH_SETUP.md
   - [ ] getting-started.md → docs/GIT_AUTH_SETUP.md
   - [ ] getting-started.md → components/README.md
   - [ ] OPENSHIFT_DEPLOY.md → docs/GIT_AUTH_SETUP.md

3. **Validate paths**:
   - [ ] All relative paths are correct from source file location
   - [ ] All target files exist
   - [ ] Links render correctly in GitHub

---

### Phase 7: Consistency Validation

**Requirements**: FR-018, FR-019, FR-021

**Steps**:

1. **Namespace consistency check**:
   ```bash
   # All examples should use ambient-code by default
   grep -r 'namespace' README.md components/README.md docs/*.md | grep -v ambient-code
   # Expected: No results (or only documented overrides)
   ```

2. **Registry consistency check**:
   ```bash
   # Pre-built images should reference quay.io/ambient_code
   grep -r 'quay.io' README.md components/README.md docs/*.md
   # Expected: All references to quay.io/ambient_code
   ```

3. **Code block language check**:
   ```bash
   # All code blocks should have language identifiers
   grep -r '```$' README.md components/README.md docs/*.md
   # Expected: No results (all blocks should be ```bash, ```yaml, etc.)
   ```

---

### Phase 8: Final Validation

**Requirements**: All FR-001 through FR-023

**Checklist**:

1. **Architecture Accuracy** (FR-001, FR-002):
   - [ ] README.md lists Next.js 15, Go 1.24, Kubernetes operator, Claude Code SDK
   - [ ] Component table has accurate technology stack
   - [ ] Versions match package.json and go.mod

2. **Deployment Documentation** (FR-003, FR-014, FR-015):
   - [ ] README.md references ./deploy.sh script
   - [ ] OPENSHIFT_DEPLOY.md documents all script options
   - [ ] Prerequisites list oc, kustomize, registry access

3. **Git Authentication** (FR-004, FR-011, FR-016):
   - [ ] README.md states git auth requirements with link
   - [ ] getting-started.md shows secret creation commands
   - [ ] OPENSHIFT_DEPLOY.md emphasizes separate secret creation

4. **Component Documentation** (FR-005, FR-006, FR-007):
   - [ ] components/README.md shows directory structure
   - [ ] Architecture flow includes 7-step session workflow
   - [ ] Quick start covers production, local dev, custom images

5. **Comprehensive Guide** (FR-008 through FR-013):
   - [ ] getting-started.md lists all prerequisites
   - [ ] Local dev setup includes CRC installation
   - [ ] Production deployment workflow is complete
   - [ ] Post-deployment steps documented
   - [ ] Troubleshooting covers common issues

6. **Consistency** (FR-018 through FR-023):
   - [ ] Namespace references consistent (ambient-code)
   - [ ] Registry references consistent (quay.io/ambient_code)
   - [ ] Cross-references use valid paths
   - [ ] Code blocks have language identifiers
   - [ ] Documentation is self-contained
   - [ ] Progressive disclosure applied

---

### Phase 9: Review and Commit

1. **Review changes**:
   ```bash
   git diff README.md
   git diff components/README.md
   git diff docs/getting-started.md
   git diff docs/OPENSHIFT_DEPLOY.md
   ```

2. **Commit changes**:
   ```bash
   git add README.md components/README.md docs/getting-started.md docs/OPENSHIFT_DEPLOY.md
   git commit -m "docs: Update vTeam documentation for current architecture

   - Update README.md with Next.js 15, Go 1.24 architecture
   - Update components/README.md with directory structure and session flow
   - Create comprehensive getting-started.md guide
   - Update OPENSHIFT_DEPLOY.md with deploy.sh options
   - Ensure consistent namespace and registry references
   - Add troubleshooting guidance

   Addresses requirements FR-001 through FR-023"
   ```

3. **Push to remote**:
   ```bash
   git push origin docs-update-001
   ```

4. **Create pull request**:
   - Title: "docs: Update vTeam documentation for current architecture"
   - Description: Reference spec.md and list all updated files
   - Reviewers: Request review from team members

---

## Validation Commands

Run these commands to verify documentation quality:

```bash
# Check for broken links (requires markdown-link-check)
npx markdown-link-check README.md
npx markdown-link-check components/README.md
npx markdown-link-check docs/getting-started.md
npx markdown-link-check docs/OPENSHIFT_DEPLOY.md

# Check markdown syntax (requires markdownlint)
npx markdownlint README.md components/README.md docs/*.md

# Verify code block languages
grep -n '```$' README.md components/README.md docs/*.md
# Should return no results

# Verify namespace consistency
grep -rn 'namespace' README.md components/README.md docs/*.md | grep -v ambient-code | grep -v NAMESPACE
# Should only show documented override examples
```

---

## Troubleshooting Implementation Issues

### Issue: Can't find vTeam repository
**Solution**: Verify repository location or clone it:
```bash
git clone https://github.com/your-org/vTeam.git
cd vTeam
```

### Issue: GIT_AUTH_SETUP.md doesn't exist
**Solution**: This file is referenced but out of scope for this RFE. Create a stub or note that it's a separate doc:
```markdown
# Git Authentication Setup
(To be created in separate documentation effort)
```

### Issue: Deploy script options don't match documentation
**Solution**: Read the actual deploy.sh script to verify options:
```bash
cat components/manifests/deploy.sh
# Look for argument parsing and usage function
```

### Issue: Component versions are outdated
**Solution**: Always read package.json and go.mod for accurate versions:
```bash
jq '.dependencies.next' components/frontend/package.json
grep '^go ' components/backend/go.mod
```

---

## Success Criteria

Documentation update is complete when:

- ✅ All 4 primary documentation files updated/created
- ✅ All functional requirements (FR-001 through FR-023) addressed
- ✅ Cross-references validated and correct
- ✅ Code blocks have language identifiers
- ✅ Namespace and registry references consistent
- ✅ Documentation renders correctly in GitHub
- ✅ Changes committed and pushed to feature branch
- ✅ Pull request created for review

---

**Generated**: 2025-10-10 by /plan command execution (Phase 1)
