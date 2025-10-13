
# Implementation Plan: Modernize FetchIt - Dependency Updates and Quadlet Method Support

**Branch**: `001-we-need-to` | **Date**: 2025-10-13 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/workspace/sessions/agentic-session-1760364724/workspace/vteam-rfes/specs/001-we-need-to/spec.md`

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

This implementation plan addresses the modernization of the FetchIt project through two primary initiatives:

**1. Dependency Updates**: Upgrade FetchIt from Go 1.17 to Go 1.22 and update Podman libraries from v4.2.0 to v4.9+. This resolves multiple high and critical severity CVEs in the Go runtime, Podman libraries, and transitive dependencies while improving performance and enabling modern language features.

**2. Quadlet Method Implementation**: Add a new "Quadlet" deployment method to FetchIt that enables declarative container management through systemd. This provides an alternative to the existing Systemd method by leveraging Podman's native Quadlet support (introduced in Podman 4.4+), which is the recommended approach for systemd integration (replacing the deprecated `podman generate systemd` command).

**Technical Approach**:
- **Go 1.22**: Target version provides optimal balance of stability, security patches (CVE-2023-45289, CVE-2024-24783/84/85), and performance improvements (40% latency reduction, 50% memory reduction for small heaps)
- **Podman v4.9+**: Final v4 release with mature Quadlet support, backwards-compatible API, and security fixes (CVE-2022-1227, CVE-2022-2989, CVE-2024-1753)
- **Quadlet Integration**: Follows existing Method interface pattern used by Raw, Systemd, Kube, Ansible, and FileTransfer methods for consistency and code reusability
- **Backwards Compatibility**: All changes are additive; existing methods and configurations continue to work without modification

**Key Benefits**:
- **Security**: Elimination of all high/critical CVEs in dependencies
- **Modern Tooling**: Quadlet is Podman's recommended systemd integration method
- **Simplified Management**: Declarative .container, .volume, and .network files managed through git
- **systemd Integration**: Full systemd service lifecycle management (enable, start, restart, stop)
- **GitOps**: Automatic synchronization of container configurations from git repositories
- **Consistency**: Quadlet method follows established FetchIt patterns for easy adoption

## Technical Context

**Language/Version**: Go 1.22 (upgrade from Go 1.17)

**Primary Dependencies**:
- `github.com/containers/podman/v4` v4.9.4 (upgrade from v4.2.0)
- `github.com/containers/common` v0.58+ (upgrade from v0.49.1)
- `github.com/containers/image/v5` v5.30+ (upgrade from v5.22.1)
- `github.com/containers/storage` v1.53+ (upgrade from v1.42.1)
- `github.com/go-git/go-git/v5` v5.12.0 (upgrade from v5.11.0)
- `github.com/go-co-op/gocron` v1.x.x (scheduling infrastructure)
- `github.com/spf13/viper` v1.18+ (configuration parsing)
- `github.com/spf13/cobra` v1.8+ (CLI framework)
- `github.com/opencontainers/runtime-spec` v1.1.0 (OCI spec)
- `go.uber.org/zap` v1.27+ (structured logging)

**Storage**: N/A - FetchIt uses existing git-based storage for configuration and state tracking via git tags. Quadlet method stores .container/.volume/.network files in systemd directories: `/etc/containers/systemd/` (root) or `~/.config/containers/systemd/` (user)

**Testing**:
- **Unit Tests**: Standard Go testing (`go test`) with table-driven tests
- **Integration Tests**: Require real Podman installation and systemd for Quadlet method
- **Test Frameworks**: Go standard library testing, testify for assertions (if used)
- **Mock Strategy**: Mock Podman API for unit tests, use real systemd/Podman for integration tests
- **CI/CD**: GitHub Actions with VM-based runners for systemd integration tests

**Target Platform**:
- **Primary**: Linux with systemd (RHEL 9+, Fedora 39+, Ubuntu 22.04+, Debian 12+)
- **Requirements**: Podman 4.4+ (4.9+ recommended), systemd, cgroup v2
- **Architectures**: AMD64 and ARM64
- **Container Runtime**: Runs as container via Podman, manages other containers via Podman API

**Project Type**: Single (Go application with cmd/ and pkg/ structure, not a web/mobile app)

**Performance Goals**:
- **Quadlet Deployment**: < 1 second for single file deployment (file copy + daemon-reload)
- **Daemon Reload**: < 5 seconds per systemctl daemon-reload operation
- **Git Sync**: < 2 seconds for typical repository size change detection
- **Concurrent Targets**: Support 10+ Quadlet targets executing concurrently
- **Schedule Overhead**: < 5% CPU, < 100MB memory per scheduled target

**Constraints**:
- **Podman Version**: Minimum 4.4+ required for Quadlet support (4.9+ recommended)
- **systemd Required**: Quadlet method only works on systemd-based Linux distributions
- **cgroup v2 Required**: Quadlet requires cgroup v2, not compatible with cgroup v1
- **Backwards Compatibility**: Must not break existing Raw, Systemd, Kube, Ansible, FileTransfer methods
- **No Breaking Config Changes**: Existing config.yaml files must continue to work
- **API Compatibility**: Must maintain compatibility with Podman v4 API (v5 migration future work)

**Scale/Scope**:
- **Quadlet Files**: Support 100+ Quadlet files per target
- **Concurrent Targets**: 10+ targets with staggered schedules
- **Repository Size**: Handle repositories with 1000+ files efficiently
- **Git History**: Work with repositories with extensive commit history
- **Scheduling**: Support high-frequency schedules (e.g., */1 * * * * - every minute)
- **Method Instances**: Multiple Quadlet method instances per TargetConfig with different targetPaths/globs

## Constitution Check
*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Constitution Status**: The constitution template at `/workspace/sessions/agentic-session-1760364724/workspace/vteam-rfes/.specify/templates/constitution-template.md` exists but is not yet populated with project-specific principles for this repository.

**Assessment**: PASS (no constitution defined)

Since no constitutional requirements have been defined for the vteam-rfes project, there are no gates to check. Once the constitution is populated with principles (e.g., simplicity rules, architectural constraints, coding standards), this design will need to be reviewed against those principles.

**Future Constitution Considerations**:
When defining the constitution for this project, consider documenting principles around:
- Backwards compatibility requirements for FetchIt
- Method interface consistency patterns
- Security and CVE management policies
- Testing coverage requirements
- Documentation standards
- Dependency update policies

## Project Structure

### Documentation (this feature)
```
specs/001-we-need-to/
├── spec.md                              # Feature specification (completed)
├── plan.md                              # This file (Phase 0-1 complete)
├── research.md                          # Phase 0 research (completed)
├── data-model.md                        # Phase 1 data model (completed)
├── quickstart.md                        # Phase 1 quickstart guide (completed)
├── contracts/                           # Phase 1 contracts (completed)
│   ├── quadlet-method-interface.md      # Method interface contract
│   ├── quadlet-config-schema.yaml       # Configuration schema
│   └── system-integration-contract.md   # systemd/Podman integration contract
└── tasks.md                             # Phase 2 output (NOT created by /plan)
```

### Source Code (repository root)

**Repository**: `/workspace/sessions/agentic-session-1760364724/workspace/fetchit/`

```
fetchit/
├── cmd/
│   └── fetchit/
│       ├── main.go                      # Application entry point
│       └── start.go                     # Start command implementation
│
├── pkg/
│   └── engine/
│       ├── types.go                     # MODIFY: Add Quadlet field to TargetConfig
│       ├── config.go                    # Configuration parsing (no changes needed)
│       ├── fetchit.go                   # MODIFY: Register Quadlet methods in getMethodTargetScheds()
│       ├── common.go                    # CommonMethod implementation (reused by Quadlet)
│       ├── apply.go                     # Change detection and application (reused)
│       ├── quadlet.go                   # NEW: Quadlet method implementation
│       ├── systemd.go                   # REFERENCE: Pattern for systemctl operations
│       ├── filetransfer.go              # REFERENCE: Pattern for file deployment
│       ├── raw.go                       # Existing Raw method (unchanged)
│       ├── kube.go                      # Existing Kube method (unchanged)
│       ├── ansible.go                   # Existing Ansible method (unchanged)
│       ├── container.go                 # Container helper functions
│       ├── image.go                     # Image operations
│       ├── clean.go                     # Cleanup operations
│       ├── gitauth.go                   # Git authentication
│       ├── disconnected.go              # Disconnected mode support
│       ├── start.go                     # Start command
│       └── utils/
│           └── utils.go                 # Utility functions
│
├── tests/
│   ├── integration/
│   │   └── quadlet_test.go              # NEW: Integration tests for Quadlet method
│   └── unit/
│       └── quadlet_test.go              # NEW: Unit tests for Quadlet method
│
├── examples/
│   └── quadlet/                         # NEW: Example Quadlet configurations
│       ├── config.yaml                  # Example FetchIt config with Quadlet
│       ├── nginx.container              # Example .container file
│       ├── nginx-data.volume            # Example .volume file
│       └── webapp.network               # Example .network file
│
├── docs/
│   ├── methods.rst                      # MODIFY: Add Quadlet method documentation
│   ├── quadlet.md                       # NEW: Detailed Quadlet method guide
│   └── migration.md                     # NEW: Migration guide (Systemd → Quadlet)
│
├── go.mod                               # MODIFY: Update Go version and dependencies
├── go.sum                               # MODIFY: Update dependency hashes
├── Dockerfile                           # MODIFY: Update base image to Go 1.22
├── Makefile                             # No changes needed
├── README.md                            # MODIFY: Update requirements, add Quadlet example
└── CHANGELOG.md                         # MODIFY: Document dependency updates and Quadlet feature
```

**Structure Decision**: Single project structure (Go application)

FetchIt is a standalone Go application with a cmd/ and pkg/ structure. It is not a web application (no frontend/backend split) or a mobile application. The project follows standard Go project layout conventions:

- **cmd/**: Application entry points and CLI commands
- **pkg/**: Reusable library code organized by domain (engine/)
- **tests/**: Test files separate from source for clarity
- **examples/**: Example configurations for users
- **docs/**: User-facing documentation

**Key Files for Quadlet Implementation**:

1. **pkg/engine/quadlet.go** (NEW): Core Quadlet method implementation
   - Implements Method interface: GetName, GetKind, GetTarget, Process, Apply, MethodEngine
   - Handles .container, .volume, .network file deployment
   - Manages systemctl daemon-reload and service lifecycle

2. **pkg/engine/types.go** (MODIFY): Add Quadlet configuration struct
   - Add `Quadlet []*Quadlet` field to TargetConfig
   - Define Quadlet struct with Root, Enable, Restart fields

3. **pkg/engine/fetchit.go** (MODIFY): Register Quadlet methods
   - Add Quadlet method registration in getMethodTargetScheds()
   - Add "quadlet" to allMethodTypes map

4. **tests/integration/quadlet_test.go** (NEW): Integration tests
   - Test deployment to systemd directories
   - Test systemctl operations
   - Test change detection and updates
   - Requires systemd and Podman 4.9+

5. **tests/unit/quadlet_test.go** (NEW): Unit tests
   - Test change type detection
   - Test destination path calculation
   - Test service name derivation
   - Mock Podman API and systemctl commands

**Dependency Updates**:
- go.mod: Update Go version from 1.17 to 1.22
- go.mod: Update Podman libraries from v4.2.0 to v4.9.4
- go.sum: Regenerated via `go mod tidy`
- Dockerfile: Update FROM golang:1.17 to FROM golang:1.22

**No Changes Required**:
- Existing methods (Raw, Systemd, Kube, Ansible, FileTransfer) unchanged
- Configuration parsing infrastructure (Viper) handles new Quadlet field automatically
- Git operations and change detection reused as-is
- Scheduling infrastructure (gocron) unchanged
- Podman API bindings usage unchanged (same v4 API)

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
   - Run `.specify/scripts/bash/update-agent-context.sh claude`
     **IMPORTANT**: Execute it exactly as specified above. Do not add or remove any arguments.
   - If exists: Add only NEW tech from current plan
   - Preserve manual additions between markers
   - Update recent changes (keep last 3)
   - Keep under 150 lines for token efficiency
   - Output to repository root

**Output**: data-model.md, /contracts/*, failing tests, quickstart.md, agent-specific file

## Phase 2: Task Planning Approach
*This section describes what the /tasks command will do - DO NOT execute during /plan*

**Task Generation Strategy**:
- Load `.specify/templates/tasks-template.md` as base
- Generate tasks from Phase 1 design docs (contracts, data model, quickstart)
- Each contract → contract test task [P]
- Each entity → model creation task [P] 
- Each user story → integration test task
- Implementation tasks to make tests pass

**Ordering Strategy**:
- TDD order: Tests before implementation 
- Dependency order: Models before services before UI
- Mark [P] for parallel execution (independent files)

**Estimated Output**: 25-30 numbered, ordered tasks in tasks.md

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
- [x] Phase 0: Research complete (/plan command) - research.md completed
- [x] Phase 1: Design complete (/plan command) - data-model.md, contracts/, quickstart.md completed
- [x] Phase 2: Task planning complete (/plan command - approach described, see Phase 2 section)
- [ ] Phase 3: Tasks generated (/tasks command - NOT executed by /plan)
- [ ] Phase 4: Implementation complete
- [ ] Phase 5: Validation passed

**Gate Status**:
- [x] Initial Constitution Check: PASS (no constitution defined, documented in Constitution Check section)
- [x] Post-Design Constitution Check: PASS (no constitution defined, no violations possible)
- [x] All NEEDS CLARIFICATION resolved (research.md addressed all technical unknowns)
- [x] Complexity deviations documented (N/A - no constitutional complexity constraints defined)

---
*Based on Constitution v2.1.1 - See `/memory/constitution.md`*
