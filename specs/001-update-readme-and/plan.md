
# Implementation Plan: Update README and Documentation for vTeam

**Branch**: `docs-vteam` | **Date**: 2025-10-10 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/workspace/sessions/agentic-session-1760129623/workspace/vteam-rfes/specs/001-update-readme-and/spec.md`

## Execution Flow (/plan command scope)
```
1. Load feature spec from Input path
   → SUCCESS: Spec loaded and analyzed
2. Fill Technical Context (scan for NEEDS CLARIFICATION)
   → Project Type: Documentation (non-code, documentation-focused)
   → Structure Decision: Single repository with documentation artifacts
3. Fill the Constitution Check section based on the content of the constitution document.
   → Constitution is template-only, no project-specific constraints apply
4. Evaluate Constitution Check section below
   → No violations detected (documentation work has minimal constitutional concerns)
   → Update Progress Tracking: Initial Constitution Check ✓
5. Execute Phase 0 → research.md
   → Analyzing existing documentation state
   → Identifying gaps between docs and code
6. Execute Phase 1 → contracts, data-model.md, quickstart.md, agent file
   → Generate documentation audit data model
   → Create documentation update contracts (what needs to be changed)
   → Create quickstart for documentation review workflow
7. Re-evaluate Constitution Check section
   → No new violations (documentation work)
   → Update Progress Tracking: Post-Design Constitution Check ✓
8. Plan Phase 2 → Describe task generation approach
9. STOP - Ready for /tasks command
```

**IMPORTANT**: The /plan command STOPS at step 7. Phases 2-4 are executed by other commands:
- Phase 2: /tasks command creates tasks.md
- Phase 3-4: Implementation execution (manual or via tools)

## Summary

This feature focuses on updating vTeam's README and documentation to accurately reflect the current implementation in the components codebase. The primary requirement is to ensure documentation completeness and accuracy by auditing and updating:

1. Main README (architecture, quick start, usage examples)
2. Component-specific READMEs (frontend, backend, operator, runners)
3. Documentation site content (user guides, developer guides, API references)
4. Configuration examples (deployment manifests, environment variables, CRDs)

The technical approach involves:
- Auditing existing documentation against component source code
- Identifying discrepancies and missing features
- Updating documentation to match actual implementation
- Ensuring all code examples, commands, and configuration are current

## Technical Context
**Language/Version**: Markdown, MkDocs static site generator, YAML (for agent personas)
**Primary Dependencies**: MkDocs, mkdocs-material theme (requirements-docs.txt)
**Storage**: Git repository (vTeam and vteam-rfes repos)
**Testing**: Manual review comparing documentation to source code, automated link checking
**Target Platform**: Documentation hosted on GitHub/GitLab Pages or similar static hosting
**Project Type**: Documentation-only (non-code artifact updates)
**Performance Goals**: N/A (documentation artifact)
**Constraints**: Must accurately reflect current implementation in components/frontend, components/backend, components/operator, components/runners
**Scale/Scope**: ~15-20 documentation files across README, docs/ directory, and component READMEs

## Constitution Check
*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Constitution Status**: Template-only constitution found at `.specify/memory/constitution.md`

Since this is a documentation-focused feature with no code implementation, and the constitution file is a template without project-specific principles, the following gates apply:

### Applicable Gates
✅ **Simplicity**: Documentation updates are straightforward - audit existing docs, identify gaps, update content
✅ **No Over-Engineering**: Using existing documentation structure (MkDocs, Markdown), no new tools or frameworks
✅ **Testability**: Documentation accuracy is testable via manual review against source code
✅ **Clear Scope**: Limited to updating existing documentation to match current implementation

### Non-Applicable Gates
- N/A: Library-first, CLI interface, TDD (this is documentation, not code)
- N/A: Integration testing, versioning (documentation follows repository versioning)

**Result**: ✅ PASS - No constitutional violations detected

## Project Structure

### Documentation (this feature)
```
specs/001-update-readme-and/
├── plan.md              # This file (/plan command output)
├── research.md          # Phase 0 output - documentation audit findings
├── data-model.md        # Phase 1 output - documentation structure model
├── quickstart.md        # Phase 1 output - documentation review workflow
├── contracts/           # Phase 1 output - documentation update contracts
└── tasks.md             # Phase 2 output (/tasks command - NOT created by /plan)
```

### Source Code (vTeam repository root)
```
vTeam/
├── README.md                            # Main project README (PRIMARY UPDATE TARGET)
├── docs/                                # Documentation site (UPDATE TARGET)
│   ├── OPENSHIFT_DEPLOY.md
│   ├── OPENSHIFT_OAUTH.md
│   ├── GITHUB_APP_SETUP.md
│   ├── CLAUDE_CODE_RUNNER.md
│   ├── user-guide/
│   ├── developer-guide/
│   └── labs/
├── components/                          # Source of truth for documentation content
│   ├── frontend/README.md               # Frontend component docs (UPDATE TARGET)
│   ├── backend/README.md                # Backend component docs (UPDATE TARGET)
│   ├── operator/README.md               # Operator component docs (UPDATE TARGET)
│   └── runners/
│       └── claude-code-runner/README.md # Runner component docs (UPDATE TARGET)
├── mkdocs.yml                           # Documentation site configuration
└── CLAUDE.md                            # Agent file (will be updated incrementally)
```

**Structure Decision**: This is a documentation-focused feature within a web application project (frontend + backend + operator + runners). The vTeam repository uses a multi-component architecture with separate directories for each service. Documentation artifacts will be updated in place within the existing repository structure. The vteam-rfes repository serves as the RFE tracking and specification repository.

## Phase 0: Outline & Research

### Research Tasks

1. **Audit Main README**
   - Extract unknowns: Which sections of README are outdated?
   - Research task: Compare README architecture section to actual components/ structure
   - Research task: Verify Quick Start commands work with current deployment scripts
   - Research task: Check if all listed components match existing implementation

2. **Audit Component Documentation**
   - Extract unknowns: What features are implemented but not documented?
   - Research task: Review frontend/README.md against frontend/src/ implementation
   - Research task: Review backend/README.md against backend/ Go packages
   - Research task: Review operator/README.md against operator/ controller logic
   - Research task: Review runners/claude-code-runner/README.md against runner implementation

3. **Audit Configuration Documentation**
   - Extract unknowns: Are all environment variables and config options documented?
   - Research task: Extract all environment variables from components/backend, components/frontend
   - Research task: Review CRD definitions in components/manifests against documentation
   - Research task: Verify deployment manifest examples are current

4. **Audit API and Integration Documentation**
   - Extract unknowns: Do API docs reflect current endpoints?
   - Research task: Compare docs/GITHUB_APP_SETUP.md to actual GitHub integration code
   - Research task: Compare docs/OPENSHIFT_OAUTH.md to actual OAuth implementation
   - Research task: Identify undocumented API endpoints in backend

5. **Audit User Guide and Developer Guide**
   - Extract unknowns: Are workflows documented correctly?
   - Research task: Test documented workflows against actual UI behavior
   - Research task: Identify missing developer setup instructions
   - Research task: Check if troubleshooting section covers actual error conditions

### Research Agent Tasks

The following research agents will be dispatched to analyze the vTeam codebase:

1. **Agent: Audit README Architecture Section**
   - Task: "Compare README architecture section to components/ directory structure and identify discrepancies"
   - Output: List of missing components, outdated descriptions, incorrect technology references

2. **Agent: Audit Component READMEs**
   - Task: "For each component (frontend, backend, operator, runners), compare README to source code and identify undocumented features"
   - Output: List of features per component that exist in code but not in docs

3. **Agent: Extract Environment Variables**
   - Task: "Scan all Go and TypeScript source files to extract environment variables and compare to documented configuration"
   - Output: List of undocumented environment variables with their purpose

4. **Agent: Audit Deployment Documentation**
   - Task: "Compare deployment documentation (OPENSHIFT_DEPLOY.md) to actual manifests and scripts in components/manifests"
   - Output: List of outdated commands, missing steps, incorrect file paths

5. **Agent: Audit API Documentation**
   - Task: "Extract all HTTP endpoints from backend Go code and compare to API documentation"
   - Output: List of undocumented or incorrectly documented API endpoints

### Consolidation Format

All findings will be consolidated in `research.md` using this format:

```markdown
## [Documentation Area]

### Decision: [What needs to be updated]
### Rationale: [Why this update is necessary - what's missing/incorrect]
### Alternatives Considered: [If applicable - e.g., restructure vs. update in place]

**Source Evidence**:
- File: [path to source file]
- Line: [line number or range]
- Current Doc: [what documentation currently says]
- Should Be: [what documentation should say based on code]
```

**Output**: research.md with all NEEDS CLARIFICATION resolved and concrete update requirements identified

## Phase 1: Design & Contracts
*Prerequisites: research.md complete*

### 1. Data Model (`data-model.md`)

Extract documentation entities from feature spec and research findings:

**Entities**:
- **DocumentationArtifact**
  - Fields: file_path, artifact_type (README/guide/reference), status (current/outdated/missing), component (frontend/backend/operator/runner/platform)
  - Relationships: References SourceCodeFile, HasUpdateRequirements
  - Validation: Must exist in vTeam repository, must have corresponding source code

- **SourceCodeFile**
  - Fields: file_path, component, features (list of features implemented in this file)
  - Relationships: DocumentedBy DocumentationArtifact
  - Validation: Must exist in components/ directory

- **DocumentationGap**
  - Fields: gap_id, artifact_path, gap_type (missing_feature/outdated_info/incorrect_example), severity (high/medium/low), source_reference (path to code that proves gap)
  - Relationships: BelongsTo DocumentationArtifact, ReferencesSourceCodeFile
  - State Transitions: identified → prioritized → fixed → verified

- **UpdateRequirement**
  - Fields: requirement_id, artifact_path, section, current_content, required_content, rationale, source_reference
  - Relationships: BelongsTo DocumentationArtifact
  - Validation: Must reference actual source code file and line number

### 2. API Contracts (`/contracts/`)

Since this is a documentation feature, "API contracts" are conceptual - they define the contract between documentation and implementation:

**Contract 1: Documentation Accuracy Contract**
```yaml
# contracts/documentation-accuracy.yaml
name: DocumentationAccuracyContract
description: Every documented feature must exist in implementation, every major feature must be documented
rules:
  - type: completeness
    constraint: All components in components/ must have corresponding documentation section
  - type: accuracy
    constraint: All code examples must reference actual files and working commands
  - type: currency
    constraint: All API endpoints documented must exist in backend/src/
  - type: configuration
    constraint: All environment variables in source must be documented with purpose
```

**Contract 2: Configuration Documentation Contract**
```yaml
# contracts/configuration-contract.yaml
name: ConfigurationDocumentationContract
description: All configuration options must be documented with examples and defaults
rules:
  - type: environment_variables
    constraint: Every env var in .env.example must be documented in README or deployment docs
  - type: deployment_manifests
    constraint: Every manifest in components/manifests must be referenced in deployment guide
  - type: crd_properties
    constraint: Every field in CRD definitions must be documented with purpose and example
```

**Contract 3: Example Code Contract**
```yaml
# contracts/example-code-contract.yaml
name: ExampleCodeContract
description: All code examples in documentation must be executable and current
rules:
  - type: commands
    constraint: Every bash command in docs must work with current scripts
  - type: api_examples
    constraint: Every API example must use current endpoint paths and parameters
  - type: configuration_examples
    constraint: Every config example must reference actual files in repo
```

### 3. Contract Tests

Since this is documentation (not code), "contract tests" are validation checklists:

**Test File: `contracts/test-readme-accuracy.md`**
```markdown
# README Accuracy Test

## Architecture Section
- [ ] Verify all components in table match components/ directories
- [ ] Verify technology stack matches package.json, go.mod
- [ ] Verify session flow steps match actual implementation in backend and operator

## Quick Start Section
- [ ] Execute deployment commands from Quick Start and verify they work
- [ ] Verify all file paths reference actual files
- [ ] Verify all oc commands produce expected output

## Configuration Section
- [ ] Verify all environment variables exist in source code
- [ ] Verify all configuration examples reference actual manifests
```

**Test File: `contracts/test-component-docs.md`**
```markdown
# Component Documentation Test

## Frontend README
- [ ] Verify described features exist in frontend/src/
- [ ] Verify dependencies match frontend/package.json
- [ ] Verify build commands work

## Backend README
- [ ] Verify described endpoints exist in backend/src/
- [ ] Verify dependencies match backend/go.mod
- [ ] Verify API examples use correct paths

## Operator README
- [ ] Verify described reconciliation logic matches operator/controllers/
- [ ] Verify CRD references match components/manifests/crds/
```

### 4. Integration Test Scenarios (`quickstart.md`)

**quickstart.md** will provide a manual test workflow for verifying documentation accuracy:

```markdown
# Documentation Review Quickstart

## Purpose
This quickstart validates that documentation accurately reflects the vTeam implementation.

## Prerequisites
- Clone vTeam repository
- Clone vteam-rfes repository
- Install: make, oc CLI, diff tools

## Validation Workflow

### Step 1: README Architecture Validation
1. Open README.md architecture section
2. List all components in README table
3. Run: `ls -1 components/`
4. Compare outputs - all directories must be documented
5. For each component, verify technology matches package.json or go.mod

### Step 2: Quick Start Command Validation
1. Follow README Quick Start exactly as written
2. Note any failing commands
3. Note any missing files or incorrect paths
4. Document all deviations

### Step 3: Component README Validation
1. For each component README:
   - Open component README
   - Open component source (src/ directory)
   - Identify 5 major features in source
   - Verify all 5 features are documented
   - Note any undocumented features

### Step 4: Configuration Documentation Validation
1. Extract all environment variables from source:
   - `grep -r "os.Getenv" components/backend/`
   - `grep -r "process.env" components/frontend/`
2. Check if each variable is documented in README or docs/
3. Note any undocumented variables

### Step 5: API Documentation Validation
1. Extract all HTTP routes from backend:
   - `grep -r "router.GET\\|router.POST" components/backend/`
2. Check if each endpoint is documented
3. Test documented API examples against actual implementation

## Expected Outcome
- All validation steps pass
- No major features are undocumented
- All examples and commands work correctly
```

### 5. Update Agent File

Run the update script to incrementally update the agent context file:

```bash
/workspace/sessions/agentic-session-1760129623/workspace/vteam-rfes/.specify/scripts/bash/update-agent-context.sh cursor
```

This will:
- Detect which agent file exists (CLAUDE.md, .github/copilot-instructions.md, etc.)
- Add only NEW technologies and patterns from this plan
- Preserve manual additions between `<!-- MANUAL CONTEXT START -->` and `<!-- MANUAL CONTEXT END -->` markers
- Update recent changes (keep last 3 entries)
- Keep file under 150 lines for token efficiency

**Output**: data-model.md, /contracts/*, validation checklists, quickstart.md, updated agent file

## Phase 2: Task Planning Approach
*This section describes what the /tasks command will do - DO NOT execute during /plan*

**Task Generation Strategy**:

1. **Load Base Template**
   - Load `.specify/templates/tasks-template.md`
   - Use execution flow defined in template

2. **Generate Audit Tasks** (from research.md findings)
   - Each DocumentationArtifact → audit task [P]
   - Task format: "Audit [artifact] against [source component]"
   - Output: List of gaps and update requirements

3. **Generate Update Tasks** (from identified gaps)
   - Each DocumentationGap (high severity) → update task
   - Each UpdateRequirement → update task
   - Task format: "Update [artifact section] to document [feature]"

4. **Generate Validation Tasks** (from contracts and quickstart)
   - Each contract → validation task
   - quickstart.md steps → manual test tasks
   - Task format: "Validate [contract] passes all checks"

5. **Generate Review Tasks**
   - Final documentation review task
   - Cross-reference validation task (ensure no broken links)

**Ordering Strategy**:

1. **TDD Order**: Audit tasks → Update tasks → Validation tasks
   - First identify what's wrong (audit)
   - Then fix it (update)
   - Then verify the fix (validate)

2. **Dependency Order**:
   - Main README before component READMEs (establishes architecture context)
   - Component READMEs before detailed guides (establishes component capabilities)
   - Configuration docs before troubleshooting (establishes what can be configured)

3. **Parallel Execution**: Mark [P] for tasks that can run in parallel
   - Audit tasks are independent [P]
   - Update tasks for different components are independent [P]
   - Validation tasks for different artifacts can run in parallel [P]

**Estimated Output**: 20-25 numbered, ordered tasks in tasks.md

**Task Categories**:
- Phase 0: Audit tasks (5-7 tasks) [P]
- Phase 1: Update main README (3-4 tasks)
- Phase 2: Update component READMEs (4-5 tasks) [P]
- Phase 3: Update guides and references (5-6 tasks) [P]
- Phase 4: Validation and review (3-4 tasks)

**IMPORTANT**: This phase is executed by the /tasks command, NOT by /plan

## Phase 3+: Future Implementation
*These phases are beyond the scope of the /plan command*

**Phase 3**: Task execution (/tasks command creates tasks.md)
**Phase 4**: Implementation (execute tasks.md - update documentation files)
**Phase 5**: Validation (run validation checklists, execute quickstart.md, verify all contracts pass)

## Complexity Tracking
*Fill ONLY if Constitution Check has violations that must be justified*

No complexity violations detected. This is a straightforward documentation update feature with no architectural complexity, no new dependencies, and no code changes.

## Progress Tracking
*This checklist is updated during execution flow*

**Phase Status**:
- [x] Phase 0: Research complete (/plan command) - ✅ research.md created
- [x] Phase 1: Design complete (/plan command) - ✅ data-model.md, contracts/, quickstart.md created
- [x] Phase 2: Task planning complete (/plan command - describe approach only) - ✅ Approach documented below
- [ ] Phase 3: Tasks generated (/tasks command) - Ready for execution
- [ ] Phase 4: Implementation complete
- [ ] Phase 5: Validation passed

**Gate Status**:
- [x] Initial Constitution Check: PASS
- [x] Post-Design Constitution Check: PASS (no violations in documentation work)
- [x] All NEEDS CLARIFICATION resolved - ✅ All resolved in research.md
- [x] Complexity deviations documented: N/A (no deviations)

**Artifacts Generated**:
- `/specs/001-update-readme-and/plan.md` - This file
- `/specs/001-update-readme-and/research.md` - Comprehensive documentation audit findings
- `/specs/001-update-readme-and/data-model.md` - Documentation artifact structure
- `/specs/001-update-readme-and/quickstart.md` - Manual validation workflow
- `/specs/001-update-readme-and/contracts/documentation-accuracy.yaml` - Accuracy contract with 8 rules
- `/specs/001-update-readme-and/contracts/test-readme-accuracy.md` - README validation checklist

---
*Based on Constitution v2.1.1 - See `.specify/memory/constitution.md`*
