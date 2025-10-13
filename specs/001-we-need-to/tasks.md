# Tasks: Modernize FetchIt - Dependency Updates and Quadlet Method Support

**Input**: Design documents from `/workspace/sessions/agentic-session-1760367173/workspace/vteam-rfes/specs/001-we-need-to/`
**Prerequisites**: plan.md, research.md, data-model.md, contracts/, quickstart.md
**Implementation Repository**: `/workspace/sessions/agentic-session-1760367173/workspace/fetchit/`

## Execution Flow
```
1. Load plan.md from feature directory ✓
   → Tech stack: Go 1.22, Podman v4.9+
   → Project structure: Single Go application (cmd/ and pkg/)
2. Load design documents: ✓
   → data-model.md: Quadlet entity extracted
   → contracts/: 3 contract files (method interface, config schema, system integration)
   → research.md: All technical decisions documented
3. Generate tasks by category:
   → Setup: Dependency updates, build verification
   → Tests: Contract tests, integration tests
   → Core: Quadlet method implementation
   → Integration: systemd, Podman API, file operations
   → Polish: Unit tests, examples, documentation
4. Task rules applied:
   → Different files = [P] for parallel execution
   → Same file = sequential (no [P])
   → Tests before implementation (TDD approach)
5. Tasks numbered T001-T038
6. Dependencies validated
7. Parallel execution examples provided
```

## Format: `[ID] [P?] Description`
- **[P]**: Can run in parallel (different files, no dependencies)
- All paths relative to repository root: `/workspace/sessions/agentic-session-1760367173/workspace/fetchit/`

---

## Phase 3.1: Setup and Dependency Updates

### T001: Update Go Version and Core Dependencies
**File**: `go.mod`
**Description**: Update Go version to 1.22 and primary Podman/container dependencies
**Actions**:
- Change `go 1.17` to `go 1.22`
- Update `github.com/containers/podman/v4` from v4.2.0 to v4.9.4
- Update `github.com/containers/common` from v0.49.1 to v0.58.0
- Update `github.com/containers/image/v5` from v5.22.1 to v5.30.0
- Update `github.com/containers/storage` from v1.42.1 to v1.53.0
- Run `go mod tidy`

### T002: Update Supporting Dependencies
**File**: `go.mod`
**Description**: Update secondary dependencies for compatibility
**Actions**:
- Update `github.com/go-git/go-git/v5` to v5.12.0
- Update `github.com/spf13/viper` to v1.18.0
- Update `github.com/spf13/cobra` to v1.8.0
- Update `go.uber.org/zap` to v1.27.0
- Update `github.com/opencontainers/runtime-spec` to v1.1.0
- Run `go mod tidy`

### T003: Update Dockerfile Base Image
**File**: `Dockerfile`
**Description**: Change base image from Go 1.17 to Go 1.22
**Actions**:
- Update `FROM golang:1.17` to `FROM golang:1.22-alpine` (or bookworm)
- Verify multi-stage build still works

### T004: Verify CVE Resolution
**Tool**: `govulncheck`
**Description**: Scan for vulnerabilities and document remaining CVEs
**Actions**:
- Install govulncheck: `go install golang.org/x/vuln/cmd/govulncheck@latest`
- Run: `govulncheck ./...`
- Document any remaining medium/low CVEs with justification
- Verify no high/critical CVEs exist

### T005: Build and Test Existing Codebase
**Files**: All existing code
**Description**: Verify backwards compatibility after dependency updates
**Actions**:
- Build: `go build -o fetchit ./cmd/fetchit`
- Run existing unit tests: `go test ./pkg/engine -v`
- Run integration tests if available
- Verify all existing methods compile without errors

---

## Phase 3.2: Tests First (TDD) ⚠️ MUST COMPLETE BEFORE 3.3

### T006: [P] Contract Test - Quadlet Method Interface
**File**: `tests/unit/quadlet_interface_test.go` (NEW)
**Description**: Test that Quadlet implements Method interface correctly
**Actions**:
- Create test file structure
- Test `GetName()` returns correct name from config
- Test `GetKind()` returns "quadlet"
- Test `GetTarget()` returns non-nil target
- All tests MUST FAIL (no implementation yet)
- Verify test runs: `go test ./tests/unit/quadlet_interface_test.go`

### T007: [P] Contract Test - Quadlet Configuration Parsing
**File**: `tests/unit/quadlet_config_test.go` (NEW)
**Description**: Test Quadlet configuration parsing from YAML
**Actions**:
- Test Quadlet struct unmarshals from config.yaml
- Test CommonMethod fields inherited correctly
- Test Root, Enable, Restart fields parse correctly
- Test invalid configurations return errors
- All tests MUST FAIL (no Quadlet struct yet)

### T008: [P] Contract Test - Change Type Detection
**File**: `tests/unit/quadlet_changes_test.go` (NEW)
**Description**: Test change type determination logic (create/update/delete/rename)
**Actions**:
- Test create: `change.From.Name == ""` && `change.To.Name != ""`
- Test update: `change.From.Name == change.To.Name`
- Test delete: `change.From.Name != ""` && `change.To.Name == ""`
- Test rename: both names different and non-empty
- All tests MUST FAIL (no MethodEngine yet)

### T009: [P] Contract Test - Destination Directory Calculation
**File**: `tests/unit/quadlet_paths_test.go` (NEW)
**Description**: Test destination path calculation for root vs user mode
**Actions**:
- Test `Root=true` → `/etc/containers/systemd/`
- Test `Root=false` → `$HOME/.config/containers/systemd/`
- Test error when `$HOME` not set in user mode
- All tests MUST FAIL (no implementation yet)

### T010: [P] Integration Test - Quadlet File Deployment
**File**: `tests/integration/quadlet_deploy_test.go` (NEW)
**Description**: Integration test for deploying .container file
**Requirements**: Podman 4.9+, systemd, cgroup v2
**Actions**:
- Create test git repository with sample .container file
- Configure Quadlet method with test config
- Execute `Process()` method
- Verify file deployed to systemd directory
- Verify systemctl daemon-reload called
- Skip if systemd not available: `t.Skip("systemd required")`
- All tests MUST FAIL (no Quadlet method yet)

### T011: [P] Integration Test - Service Lifecycle
**File**: `tests/integration/quadlet_lifecycle_test.go` (NEW)
**Description**: Integration test for create → update → delete lifecycle
**Requirements**: Podman 4.9+, systemd, cgroup v2
**Actions**:
- Deploy .container file (create)
- Verify service starts
- Update .container file (update)
- Verify service restarts (if Restart=true)
- Delete .container file (delete)
- Verify service stops and file removed
- Skip if systemd not available
- All tests MUST FAIL (no implementation yet)

### T012: [P] Integration Test - Multiple File Types
**File**: `tests/integration/quadlet_multi_test.go` (NEW)
**Description**: Test deployment of .container, .volume, and .network files
**Requirements**: Podman 4.9+, systemd, cgroup v2
**Actions**:
- Create test repo with .container, .volume, .network files
- Deploy all three types
- Verify volume created: `podman volume ls`
- Verify network created: `podman network ls`
- Verify container uses volume and network
- Skip if systemd not available
- All tests MUST FAIL (no implementation yet)

---

## Phase 3.3: Core Implementation (ONLY after tests are failing)

### T013: Add Quadlet Struct to TargetConfig
**File**: `pkg/engine/types.go`
**Description**: Extend TargetConfig with Quadlet field
**Actions**:
- Add `Quadlet []*Quadlet \`mapstructure:"quadlet"\`` field to TargetConfig struct
- Define Quadlet struct with CommonMethod embedding
- Add `Root bool \`mapstructure:"root"\``
- Add `Enable bool \`mapstructure:"enable"\``
- Add `Restart bool \`mapstructure:"restart"\``
- Verify tests T006, T007 pass after this change

### T014: Register Quadlet Method in Engine
**File**: `pkg/engine/fetchit.go`
**Description**: Register Quadlet methods in getMethodTargetScheds()
**Actions**:
- Add `quadletMethod = "quadlet"` constant
- Add registration block for Quadlet in getMethodTargetScheds():
  ```go
  if len(tc.Quadlet) > 0 {
      fetchit.allMethodTypes[quadletMethod] = struct{}{}
      for _, q := range tc.Quadlet {
          q.initialRun = true
          q.target = internalTarget
          fetchit.methodTargetScheds[q] = q.SchedInfo()
      }
  }
  ```

### T015: Implement Quadlet GetName, GetKind, GetTarget
**File**: `pkg/engine/quadlet.go` (NEW)
**Description**: Implement basic Method interface methods
**Actions**:
- Create new file `pkg/engine/quadlet.go`
- Implement `GetName() string` - return q.Name
- Implement `GetKind() string` - return "quadlet"
- Implement `GetTarget() *Target` - return q.target
- Verify tests T006 pass

### T016: Implement Quadlet Process Method
**File**: `pkg/engine/quadlet.go`
**Description**: Implement main Process() entry point following Systemd pattern
**Actions**:
- Implement `Process(ctx, conn context.Context, skew int)` method
- Sleep for skew milliseconds
- Acquire target mutex lock
- Handle initialRun: call getRepo() and zeroToCurrent()
- Handle updates: call currentToLatest()
- Release mutex
- Set tags: `[]string{".container", ".volume", ".network"}`
- Handle Restart implies Enable logic
- Pattern: Follow pkg/engine/systemd.go:Process() implementation

### T017: Implement Quadlet Apply Method
**File**: `pkg/engine/quadlet.go`
**Description**: Implement Apply() to compute and apply git changes
**Actions**:
- Implement `Apply(ctx, conn context.Context, currentState, desiredState plumbing.Hash, tags *[]string) error`
- Call `applyChanges()` to compute diff and filter files
- Call `runChanges()` to iterate and invoke MethodEngine()
- Return error if any deployment fails
- Pattern: Follow pkg/engine/systemd.go:Apply() implementation

### T018: Implement Quadlet MethodEngine (Change Detection)
**File**: `pkg/engine/quadlet.go`
**Description**: Implement MethodEngine() for change type determination and delegation
**Actions**:
- Implement `MethodEngine(ctx, conn context.Context, change *object.Change, path string) error`
- Determine change type: create/update/rename/delete
- Extract prev and curr filenames from change object
- Determine destination directory based on Root flag
- Check $HOME environment variable for user mode
- Call `quadletPodman()` helper with all parameters
- Verify tests T008, T009 pass

### T019: Implement quadletPodman Helper (File Operations)
**File**: `pkg/engine/quadlet.go`
**Description**: Implement file deployment using FileTransfer pattern
**Actions**:
- Implement `quadletPodman(ctx, conn context.Context, path, dest string, prev, curr *string, changeType *string) error`
- For create/update: Copy file from git to systemd directory
  - Reuse FileTransfer method pattern from pkg/engine/filetransfer.go
  - Ensure destination directory exists
  - Set file permissions to 0644
- For delete: Remove file from systemd directory
- For rename: Delete old file, create new file
- Call `systemctlDaemonReload()` after file changes
- Call `systemctlManageService()` for service lifecycle
- Add comprehensive error logging

### T020: Implement systemctlDaemonReload Helper
**File**: `pkg/engine/quadlet.go`
**Description**: Execute systemctl daemon-reload via helper container
**Actions**:
- Implement `systemctlDaemonReload(conn context.Context, dest string, root bool) error`
- Create privileged helper container (pattern from Systemd method)
- Mount systemd directories: `/run/systemd/`, `/sys/fs/cgroup/`
- Mount Quadlet directory: dest parameter
- Execute `systemctl daemon-reload` (root) or `systemctl --user daemon-reload` (user)
- Wait for completion, capture exit code
- Remove helper container
- Return error if exit code non-zero
- Reference: pkg/engine/systemd.go:enableRestartSystemdService()

### T021: Implement systemctlManageService Helper
**File**: `pkg/engine/quadlet.go`
**Description**: Execute systemctl commands for service management
**Actions**:
- Implement `systemctlManageService(conn context.Context, action, service string, root bool) error`
- Create privileged helper container
- Mount systemd socket and cgroup filesystem
- Execute `systemctl {action} {service}` or `systemctl --user {action} {service}`
- Supported actions: enable, start, restart, stop, disable
- Capture stdout/stderr for error reporting
- Remove helper container
- Return error if exit code non-zero
- Derive service name: `filename.container` → `filename.service`

### T022: Implement Environment Validation
**File**: `pkg/engine/quadlet.go`
**Description**: Validate runtime environment for Quadlet support
**Actions**:
- Implement `validateEnvironment(conn context.Context) error` method
- Check Podman version >= 4.4.0 via `system.Version()`
- Check cgroup v2 via `system.Info()` - `Host.CgroupsVersion == "v2"`
- Check systemd availability (attempt systemctl --version)
- Check directory writability based on Root flag
- Return clear error messages for each failure
- Call this validation in Process() before first deployment

---

## Phase 3.4: Integration and Configuration

### T023: Create Example Quadlet Configuration
**File**: `examples/quadlet/config.yaml` (NEW)
**Description**: Create example FetchIt configuration with Quadlet
**Actions**:
- Create directory: `examples/quadlet/`
- Create config.yaml with:
  - Git URL (test repository or local)
  - Quadlet method configuration
  - Root and user mode examples
  - Enable and restart settings
  - Schedule configuration

### T024: [P] Create Example .container File
**File**: `examples/quadlet/nginx.container` (NEW)
**Description**: Create sample Nginx container Quadlet file
**Actions**:
- Create nginx.container with:
  - [Unit] section: Description, After, Wants
  - [Container] section: Image, PublishPort, Volume, Environment
  - [Service] section: Restart policy
  - [Install] section: WantedBy targets
- Document inline with comments

### T025: [P] Create Example .volume File
**File**: `examples/quadlet/nginx-data.volume` (NEW)
**Description**: Create sample volume Quadlet file
**Actions**:
- Create nginx-data.volume with:
  - [Unit] section: Before nginx.service
  - [Volume] section: Labels
  - [Install] section: WantedBy targets

### T026: [P] Create Example .network File
**File**: `examples/quadlet/webapp.network` (NEW)
**Description**: Create sample network Quadlet file
**Actions**:
- Create webapp.network with:
  - [Unit] section: Before container services
  - [Network] section: Subnet, Gateway, Labels
  - [Install] section: WantedBy targets

### T027: Update Configuration Parsing
**File**: `pkg/engine/config.go`
**Description**: Verify Viper parses Quadlet configuration correctly
**Actions**:
- Test configuration loading with Quadlet sections
- Verify no code changes needed (mapstructure handles it)
- Add validation for Quadlet-specific fields if needed
- Document any configuration edge cases

---

## Phase 3.5: Documentation and Polish

### T028: [P] Create Quadlet Method Documentation
**File**: `docs/quadlet.md` (NEW)
**Description**: Detailed user-facing Quadlet documentation
**Actions**:
- Introduction: What is Quadlet, why use it
- Prerequisites: Podman 4.4+, systemd, cgroup v2
- Configuration examples: Root and user modes
- File format reference: .container, .volume, .network
- Common patterns: Multi-container apps, volumes, networks
- Troubleshooting: Common errors and solutions
- Migration guide: From Systemd method to Quadlet

### T029: [P] Update Main Methods Documentation
**File**: `docs/methods.rst`
**Description**: Add Quadlet to methods documentation
**Actions**:
- Add Quadlet section after Systemd method
- Link to docs/quadlet.md for details
- Update method comparison table
- Document when to use Quadlet vs other methods

### T030: [P] Create Migration Guide
**File**: `docs/migration.md` (NEW)
**Description**: Guide for migrating from Systemd to Quadlet method
**Actions**:
- Explain differences between methods
- Step-by-step migration process
- Configuration conversion examples
- Testing and validation steps
- Rollback procedures

### T031: [P] Update README.md
**File**: `README.md`
**Description**: Update README with Quadlet support and new requirements
**Actions**:
- Update minimum requirements: Go 1.22, Podman 4.9+
- Add Quadlet to features list
- Add Quadlet example in quick start section
- Update build instructions if needed
- Update compatibility matrix

### T032: [P] Update CHANGELOG.md
**File**: `CHANGELOG.md`
**Description**: Document all changes in this feature
**Actions**:
- Add new version section (e.g., v0.X.0)
- Document dependency updates:
  - Go 1.17 → 1.22
  - Podman v4.2.0 → v4.9.4
  - All other dependency updates
- Document new Quadlet method feature
- Document security improvements (CVE fixes)
- Document any breaking changes (if any)

### T033: [P] Add Unit Tests for Quadlet Helpers
**File**: `tests/unit/quadlet_helpers_test.go` (NEW)
**Description**: Unit tests for helper functions
**Actions**:
- Test service name derivation from filenames
- Test destination directory calculation edge cases
- Test change type logic with various inputs
- Test error messages format
- Mock systemctl and Podman calls
- Aim for 80%+ code coverage

### T034: [P] Add Error Handling Tests
**File**: `tests/unit/quadlet_errors_test.go` (NEW)
**Description**: Test error handling and recovery
**Actions**:
- Test behavior with missing $HOME
- Test behavior with non-writable directories
- Test behavior with Podman version too old
- Test behavior with cgroup v1
- Test behavior with systemd not available
- Test behavior with malformed Quadlet files
- Verify error messages are clear and actionable

### T035: Create Quickstart Validation Script
**File**: `tests/quickstart-validation.sh` (NEW)
**Description**: Automated script to run quickstart guide steps
**Actions**:
- Script to execute all quickstart.md steps
- Verify each success criteria
- Generate report of passing/failing steps
- Use for CI/CD validation

---

## Phase 3.6: Final Verification

### T036: Run Full Test Suite
**Files**: All test files
**Description**: Execute all unit and integration tests
**Actions**:
- Run: `go test ./... -v`
- Verify all tests pass
- Generate coverage report: `go test ./... -coverprofile=coverage.out`
- Verify coverage >= 70% for new Quadlet code
- Fix any failing tests

### T037: Build Multi-Architecture Binaries
**Tool**: `go build`
**Description**: Build for AMD64 and ARM64
**Actions**:
- Build AMD64: `GOARCH=amd64 go build -o fetchit-amd64 ./cmd/fetchit`
- Build ARM64: `GOARCH=arm64 go build -o fetchit-arm64 ./cmd/fetchit`
- Test basic functionality: `./fetchit-amd64 --version`
- Verify binary sizes reasonable

### T038: Execute Full Quickstart Guide
**File**: `specs/001-we-need-to/quickstart.md`
**Description**: Manually execute all quickstart steps end-to-end
**Actions**:
- Follow each step in quickstart.md
- Verify all success criteria met
- Test on clean VM with Podman 4.9+
- Test both root and user modes
- Document any deviations or issues
- Update quickstart if needed

---

## Dependencies

### Phase Order
- **Phase 3.1** (Setup) → Must complete before all other phases
- **Phase 3.2** (Tests) → Must complete before Phase 3.3
- **Phase 3.3** (Core) → Must complete before Phase 3.4, 3.5
- **Phase 3.4** (Integration) → Can overlap with Phase 3.5
- **Phase 3.5** (Documentation) → Can start once Phase 3.3 is stable
- **Phase 3.6** (Verification) → Must be last

### Task Dependencies
- T001, T002 → T003 (dependencies before Dockerfile)
- T001, T002 → T004 (dependencies before CVE check)
- T004 → T005 (CVE check before build test)
- T005 → T006-T012 (build works before writing tests)
- T006-T012 → T013 (tests fail before implementation)
- T013 → T014 (types before registration)
- T014 → T015 (registration before implementation)
- T015 → T016, T017, T018 (basic methods before Process/Apply/MethodEngine)
- T018 → T019, T020, T021 (MethodEngine before helpers)
- T019, T020, T021 → T022 (helpers before validation)
- T022 → T023-T027 (validation before examples)
- T023-T027 → T028-T032 (examples before docs)
- T013-T022 → T033, T034 (implementation before unit tests)
- T028-T034 → T035 (docs and tests before validation script)
- T001-T035 → T036, T037, T038 (everything before final verification)

### Blocking Relationships
- **T013 blocks**: T014, T015 (types needed for registration)
- **T016 blocks**: T010, T011, T012 (Process needed for integration tests)
- **T019 blocks**: T010, T011, T012 (file ops needed for integration tests)
- **T020, T021 block**: T011, T012 (systemctl needed for lifecycle tests)

---

## Parallel Execution Examples

### Phase 3.1: Setup (Sequential)
```
# Must run in order due to same file (go.mod)
T001 → T002 → T003 → T004 → T005
```

### Phase 3.2: Test Creation (Parallel)
```
# All create different test files, can run in parallel
Task: "tests/unit/quadlet_interface_test.go"
Task: "tests/unit/quadlet_config_test.go"
Task: "tests/unit/quadlet_changes_test.go"
Task: "tests/unit/quadlet_paths_test.go"
Task: "tests/integration/quadlet_deploy_test.go"
Task: "tests/integration/quadlet_lifecycle_test.go"
Task: "tests/integration/quadlet_multi_test.go"
```

### Phase 3.3: Core Implementation (Mostly Sequential)
```
# Same file (quadlet.go), must be sequential
T013 → T014 → T015 → T016 → T017 → T018 → T019 → T020 → T021 → T022
```

### Phase 3.4: Examples (Parallel)
```
# All create different files, can run in parallel
Task: "examples/quadlet/nginx.container"
Task: "examples/quadlet/nginx-data.volume"
Task: "examples/quadlet/webapp.network"
```

### Phase 3.5: Documentation (Parallel)
```
# All different files, can run in parallel
Task: "docs/quadlet.md"
Task: "docs/methods.rst"
Task: "docs/migration.md"
Task: "README.md"
Task: "CHANGELOG.md"
Task: "tests/unit/quadlet_helpers_test.go"
Task: "tests/unit/quadlet_errors_test.go"
```

---

## Notes

### TDD Approach
- Phase 3.2 tests **MUST FAIL** before starting Phase 3.3
- Verify each test fails with "undefined" or "not implemented" errors
- Implement code to make tests pass one by one
- Commit after each test passes

### File Modification Tracking
- **Single file, sequential**: go.mod (T001, T002), quadlet.go (T013-T022), types.go (T013), fetchit.go (T014)
- **Multiple files, parallel**: All test files, all example files, all doc files
- **No conflicts**: Tests use different files, docs use different files

### Success Criteria
- All 38 tasks completed
- All tests passing (unit and integration)
- CVE scan shows no high/critical vulnerabilities
- Both AMD64 and ARM64 binaries build successfully
- Quickstart guide executes without errors
- Documentation reviewed and accurate

### Risk Mitigation
- Incremental testing after each phase
- Rollback capability (git branches for each phase)
- Existing methods must continue working (backwards compatibility)
- Clear error messages for unsupported environments

---

## Validation Checklist
*GATE: Must verify before marking complete*

- [x] All contracts have corresponding tests (T006-T012)
- [x] All entities have implementation tasks (Quadlet entity → T013-T022)
- [x] All tests come before implementation (Phase 3.2 before 3.3)
- [x] Parallel tasks truly independent (different files)
- [x] Each task specifies exact file path
- [x] No task modifies same file as another [P] task
- [x] Dependencies validated and documented
- [x] Quickstart scenarios covered in T038
- [x] All functional requirements mapped to tasks

---

**Total Tasks**: 38
**Estimated Effort**: 15-20 development days
**Parallelization Potential**: ~30% of tasks can run in parallel
**Critical Path**: T001 → T002 → T005 → T006 → T013 → T015 → T016 → T019 → T036 → T038
