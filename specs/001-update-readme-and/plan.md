
# Implementation Plan: Update README and Other Docs for vTeam

**Branch**: `001-update-readme-and` | **Date**: 2025-10-10 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/workspace/sessions/agentic-session-1760134354/workspace/vteam-rfes/specs/001-update-readme-and/spec.md`

## Execution Flow (/plan command scope)
```
1. Load feature spec from Input path
   → If not found: ERROR "No feature spec at {path}"
2. Fill Technical Context (scan for NEEDS CLARIFICATION)
   → Detect Project Type from file system structure or context (web=frontend+backend, mobile=app+api)
   → Set Structure Decision based on project type
3. Fill the Constitution Check section based on the content of the constitution document.
4. Evaluate Constitution Check section below
   → If violations exist: Document in Complexity Tracking
   → If no justification possible: ERROR "Simplify approach first"
   → Update Progress Tracking: Initial Constitution Check
5. Execute Phase 0 → research.md
   → If NEEDS CLARIFICATION remain: ERROR "Resolve unknowns"
6. Execute Phase 1 → contracts, data-model.md, quickstart.md, agent-specific template file (e.g., `CLAUDE.md` for Claude Code, `.github/copilot-instructions.md` for GitHub Copilot, `GEMINI.md` for Gemini CLI, `QWEN.md` for Qwen Code or `AGENTS.md` for opencode).
7. Re-evaluate Constitution Check section
   → If new violations: Refactor design, return to Phase 1
   → Update Progress Tracking: Post-Design Constitution Check
8. Plan Phase 2 → Describe task generation approach (DO NOT create tasks.md)
9. STOP - Ready for /tasks command
```

**IMPORTANT**: The /plan command STOPS at step 7. Phases 2-4 are executed by other commands:
- Phase 2: /tasks command creates tasks.md
- Phase 3-4: Implementation execution (manual or via tools)

## Summary
Update vTeam platform documentation to accurately reflect current implementation: Next.js 15 frontend (React + Shadcn UI), Go backend (Gin framework), Kubernetes operator, and Claude Code SDK runner. Comprehensive documentation updates include README.md, components/README.md, getting-started.md, and OPENSHIFT_DEPLOY.md. Documentation must provide clear deployment instructions using the `components/manifests/deploy.sh` script, explain git authentication configuration per project, and enable users to successfully set up local development and production deployments.

## Technical Context
**Language/Version**: Markdown documentation files (no code implementation)
**Primary Dependencies**: N/A - Documentation updates only
**Storage**: Markdown files in vTeam repository (README.md, components/README.md, docs/getting-started.md, docs/OPENSHIFT_DEPLOY.md)
**Testing**: Manual verification of documentation accuracy against source code and deployment scripts
**Target Platform**: GitHub-rendered markdown (viewable in repository browser and locally)
**Project Type**: Documentation-only - no source code structure changes
**Performance Goals**: N/A - Documentation accessibility and clarity
**Constraints**: Documentation must accurately reflect existing codebase; no code changes allowed
**Scale/Scope**: 4 primary documentation files, ~23 functional requirements covering architecture, deployment, configuration, and troubleshooting

**Additional Context from vTeam Source Repository**:
- **Frontend**: Next.js 15.5.2, React 19.1.0, TypeScript, Shadcn UI components (Radix UI), Tailwind CSS 4
- **Backend**: Go 1.24, Gin framework 1.10.1, Kubernetes client-go 0.34.0, WebSocket support (Gorilla)
- **Component Structure**: `/workspace/.../vTeam/components/` with subdirectories: backend/, frontend/, operator/, runners/, manifests/, scripts/
- **Deployment**: `components/manifests/deploy.sh` script with options for standard deploy, custom namespace, secrets-only mode, and uninstall
- **Target Documentation in vTeam Repository**: `/workspace/.../vTeam/README.md`, `/workspace/.../vTeam/components/README.md`, `/workspace/.../vTeam/docs/getting-started.md`, `/workspace/.../vTeam/docs/OPENSHIFT_DEPLOY.md`

## Constitution Check
*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Constitution Status**: The constitution file at `.specify/memory/constitution.md` is a template and has not been filled out with project-specific principles. This is a documentation-only feature with no code implementation, so constitutional principles around library-first architecture, CLI interfaces, and TDD do not apply.

**Documentation-Specific Principles Applied**:
- ✅ **Accuracy**: Documentation must accurately reflect current codebase implementation
- ✅ **Completeness**: All user scenarios from spec (local dev, production deploy, git auth) must be covered
- ✅ **Consistency**: Cross-references, commands, and namespace references must be consistent across all docs
- ✅ **Clarity**: Progressive disclosure (quick start first, detailed configuration later)
- ✅ **No Code Changes**: This feature is strictly documentation updates; no source code modifications

**Violations**: None - Documentation updates do not introduce architectural complexity

## Project Structure

### Documentation (this feature)
```
specs/[###-feature]/
├── plan.md              # This file (/plan command output)
├── research.md          # Phase 0 output (/plan command)
├── data-model.md        # Phase 1 output (/plan command)
├── quickstart.md        # Phase 1 output (/plan command)
├── contracts/           # Phase 1 output (/plan command)
└── tasks.md             # Phase 2 output (/tasks command - NOT created by /plan)
```

### Source Code (vTeam repository - target for documentation updates)
```
vTeam/                          # Main vTeam repository
├── README.md                   # [UPDATE] Root documentation - architecture, quick start
├── components/
│   ├── README.md              # [UPDATE] Component overview and quick start
│   ├── backend/               # Go backend source (Gin framework)
│   ├── frontend/              # Next.js 15 frontend source (React, Shadcn UI)
│   ├── operator/              # Kubernetes operator source
│   ├── runners/               # Claude Code SDK runner source
│   ├── manifests/             # Kubernetes manifests and deploy.sh script
│   └── scripts/               # Build and utility scripts
├── docs/
│   ├── getting-started.md     # [CREATE] Comprehensive setup guide
│   ├── OPENSHIFT_DEPLOY.md    # [UPDATE] Deployment guide with deploy.sh details
│   └── GIT_AUTH_SETUP.md      # [REFERENCE] Git authentication configuration
└── Makefile                   # Build and deployment targets
```

**Structure Decision**: This is a documentation-only feature targeting the vTeam repository located at `/workspace/sessions/agentic-session-1760134354/workspace/vTeam/`. No source code changes will be made. Documentation files will be created or updated in the vTeam repository, not in the vteam-rfes RFE repository. The implementation will involve reading the existing codebase to extract accurate information, then writing/updating markdown files.

## Phase 0: Outline & Research
1. **Extract unknowns from Technical Context** above:
   - For each NEEDS CLARIFICATION → research task
   - For each dependency → best practices task
   - For each integration → patterns task

2. **Generate and dispatch research agents**:
   ```
   For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"
   ```

3. **Consolidate findings** in `research.md` using format:
   - Decision: [what was chosen]
   - Rationale: [why chosen]
   - Alternatives considered: [what else evaluated]

**Output**: research.md with all NEEDS CLARIFICATION resolved

## Phase 1: Design & Contracts
*Prerequisites: research.md complete*

1. **Extract entities from feature spec** → `data-model.md`:
   - Entity name, fields, relationships
   - Validation rules from requirements
   - State transitions if applicable

2. **Generate API contracts** from functional requirements:
   - For each user action → endpoint
   - Use standard REST/GraphQL patterns
   - Output OpenAPI/GraphQL schema to `/contracts/`

3. **Generate contract tests** from contracts:
   - One test file per endpoint
   - Assert request/response schemas
   - Tests must fail (no implementation yet)

4. **Extract test scenarios** from user stories:
   - Each story → integration test scenario
   - Quickstart test = story validation steps

5. **Update agent file incrementally** (O(1) operation):
   - Run `.specify/scripts/bash/update-agent-context.sh cursor`
     **IMPORTANT**: Execute it exactly as specified above. Do not add or remove any arguments.
   - If exists: Add only NEW tech from current plan
   - Preserve manual additions between markers
   - Update recent changes (keep last 3)
   - Keep under 150 lines for token efficiency
   - Output to repository root

**Output**: data-model.md, /contracts/*, failing tests, quickstart.md, agent-specific file

## Phase 2: Task Planning Approach
*This section describes what the /tasks command will do - DO NOT execute during /plan*

**Task Generation Strategy for Documentation Feature**:

Since this is a documentation-only feature, the task structure differs from traditional code implementation:

1. **Load `.specify/templates/tasks-template.md` as base**
2. **Generate tasks from Phase 1 artifacts**:
   - research.md → Research verification tasks
   - data-model.md → Documentation structure validation tasks
   - contracts/*.md → Documentation file creation/update tasks (one task per contract)
   - quickstart.md → Validation and review tasks

3. **Task Categories**:
   - **Preparation Tasks**: Setup vTeam repository, create feature branch
   - **File Update Tasks**: One task per documentation file (README.md, components/README.md, getting-started.md, OPENSHIFT_DEPLOY.md)
   - **Validation Tasks**: Cross-reference validation, consistency checks, syntax validation
   - **Review Tasks**: Final review against contracts, commit and push

4. **Ordering Strategy**:
   - Preparation first (setup environment)
   - Core files in dependency order:
     1. README.md (root-level overview, referenced by others)
     2. components/README.md (component details, referenced by getting-started)
     3. getting-started.md (comprehensive guide, references others)
     4. OPENSHIFT_DEPLOY.md (deployment-specific, references getting-started)
   - Validation after all files updated
   - Review and commit last

5. **Parallelization**:
   - Research verification can run in parallel [P]
   - File updates are sequential (dependencies via cross-references)
   - Validation tasks can run in parallel after all files updated [P]

**Estimated Output**: 15-20 numbered, ordered tasks in tasks.md

**Task Breakdown Example**:
1. [P] Verify research findings against vTeam codebase
2. Setup: Navigate to vTeam repository and create feature branch
3. Update README.md per contract (FR-001 through FR-004)
4. Update components/README.md per requirements (FR-005 through FR-007)
5. Create docs/getting-started.md per contract (FR-008 through FR-013)
6. Update docs/OPENSHIFT_DEPLOY.md per requirements (FR-014 through FR-017)
7. [P] Validate cross-references (FR-020)
8. [P] Validate namespace consistency (FR-018)
9. [P] Validate registry consistency (FR-019)
10. [P] Validate code block syntax (FR-021)
11. Final review against all contracts
12. Commit changes with descriptive message
13. Push to remote and create pull request

**IMPORTANT**: This phase is executed by the /tasks command, NOT by /plan

## Phase 3+: Future Implementation
*These phases are beyond the scope of the /plan command*

**Phase 3**: Task execution (/tasks command creates tasks.md)  
**Phase 4**: Implementation (execute tasks.md following constitutional principles)  
**Phase 5**: Validation (run tests, execute quickstart.md, performance validation)

## Complexity Tracking
*Fill ONLY if Constitution Check has violations that must be justified*

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |


## Progress Tracking
*This checklist is updated during execution flow*

**Phase Status**:
- [x] Phase 0: Research complete (/plan command) - research.md created
- [x] Phase 1: Design complete (/plan command) - data-model.md, contracts/, quickstart.md created
- [x] Phase 2: Task planning complete (/plan command - approach described)
- [ ] Phase 3: Tasks generated (/tasks command)
- [ ] Phase 4: Implementation complete
- [ ] Phase 5: Validation passed

**Gate Status**:
- [x] Initial Constitution Check: PASS (documentation-only feature, constitutional principles N/A)
- [x] Post-Design Constitution Check: PASS (no architectural complexity introduced)
- [x] All NEEDS CLARIFICATION resolved (no technical unknowns, all info in codebase)
- [x] Complexity deviations documented (N/A - no deviations)

**Artifacts Generated**:
- [x] research.md - Technology stack and deployment research findings
- [x] data-model.md - Documentation entity definitions and relationships
- [x] contracts/README-contract.md - Requirements for README.md updates
- [x] contracts/getting-started-contract.md - Requirements for getting-started.md creation
- [x] quickstart.md - Implementation workflow for documentation updates
- [x] .cursor/rules/specify-rules.mdc - Updated agent context file

**Ready for /tasks Command**: ✅ Yes - All Phase 0 and Phase 1 artifacts complete

---
*Based on Constitution v2.1.1 - See `/memory/constitution.md`*
