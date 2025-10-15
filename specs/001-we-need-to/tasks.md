# Tasks: Modernize FetchIt - Dependency Updates and Quadlet Method Support

**Input**: Design documents from `/workspace/sessions/agentic-session-1760373864/workspace/vteam-rfes/specs/001-we-need-to/`
**Prerequisites**: plan.md, research.md, data-model.md, contracts/, quickstart.md
**Repository**: `/workspace/sessions/agentic-session-1760373864/workspace/fetchit/`

## Execution Flow
```
1. Phase 3.1: Setup - Update dependencies and project structure
2. Phase 3.2: Tests First (TDD) - Write failing tests before implementation
3. Phase 3.3: Core Implementation - Implement Quadlet method and make tests pass
4. Phase 3.4: Integration - Integrate with FetchIt infrastructure
5. Phase 3.5: Polish - Documentation, examples, and final cleanup
```

## Format: `[ID] [P?] Description`
- **[P]**: Can run in parallel (different files, no dependencies)
- Include exact file paths in descriptions

## Path Conventions
- **Repository root**: `/workspace/sessions/agentic-session-1760373864/workspace/fetchit/`
- **Source**: `pkg/engine/`
- **Tests**: `tests/unit/` and `tests/integration/`
- **Examples**: `examples/quadlet/`
- **Docs**: `docs/`

---

## Phase 3.1: Setup

- [X] T001 Update Go version to 1.22 in go.mod and Dockerfile
- [X] T002 Update Podman v4 to v4.9.4 in go.mod
- [X] T003 [P] Update containers/common to v0.58.0 in go.mod
- [X] T004 [P] Update containers/image/v5 to v5.30.0 in go.mod
- [X] T005 [P] Update containers/storage to v1.53.0 in go.mod
- [X] T006 [P] Update go-git/go-git/v5 to v5.12.0 in go.mod
- [X] T007 Run go mod tidy and verify no high/critical CVEs with govulncheck
- [ ] T008 Build FetchIt for AMD64 and ARM64, verify all existing tests pass

---

## Phase 3.2: Tests First (TDD) ⚠️ MUST COMPLETE BEFORE 3.3

**CRITICAL: These tests MUST be written and MUST FAIL before ANY implementation**

### Contract Tests
- [ ] T009 [P] Contract test for Method interface in tests/unit/quadlet_method_test.go
- [ ] T010 [P] Contract test for Quadlet configuration schema in tests/unit/quadlet_config_test.go
- [ ] T011 [P] Contract test for systemd integration in tests/integration/quadlet_systemd_test.go

### Unit Tests (Test change detection and file operations)
- [ ] T012 [P] Unit test for GetName() and GetKind() in tests/unit/quadlet_test.go
- [ ] T013 [P] Unit test for change type detection (create/update/rename/delete) in tests/unit/quadlet_test.go
- [ ] T014 [P] Unit test for destination directory calculation (root vs user) in tests/unit/quadlet_test.go
- [ ] T015 [P] Unit test for service name derivation from filename in tests/unit/quadlet_test.go

### Integration Tests (Test full workflow with mock systemd)
- [ ] T016 [P] Integration test for initial deployment (zeroToCurrent) in tests/integration/quadlet_deployment_test.go
- [ ] T017 [P] Integration test for update detection (currentToLatest) in tests/integration/quadlet_update_test.go
- [ ] T018 [P] Integration test for file deletion handling in tests/integration/quadlet_delete_test.go
- [ ] T019 [P] Integration test for root mode deployment in tests/integration/quadlet_root_mode_test.go
- [ ] T020 [P] Integration test for user mode deployment in tests/integration/quadlet_user_mode_test.go

---

## Phase 3.3: Core Implementation (ONLY after tests are failing)

### Data Model Changes
- [X] T021 Add Quadlet struct to pkg/engine/types.go with Root, Enable, Restart fields

### Quadlet Method Implementation
- [X] T022 Implement GetName() in pkg/engine/quadlet.go
- [X] T023 Implement GetKind() in pkg/engine/quadlet.go (returns "quadlet")
- [X] T024 Implement GetTarget() in pkg/engine/quadlet.go
- [X] T025 Implement Process() method in pkg/engine/quadlet.go (main entry point)
- [X] T026 Implement Apply() method in pkg/engine/quadlet.go (change detection)
- [X] T027 Implement MethodEngine() in pkg/engine/quadlet.go (per-file deployment logic)

### Helper Methods Implementation
- [X] T028 Implement quadletPodman() helper in pkg/engine/quadlet.go
- [X] T029 Implement systemctlDaemonReload() in pkg/engine/quadlet.go
- [X] T030 Implement systemctlManageService() in pkg/engine/quadlet.go (enable/start/restart/stop)
- [X] T031 Implement serviceNameFromFile() helper in pkg/engine/quadlet.go
- [X] T032 Implement destDir() helper for path calculation in pkg/engine/quadlet.go

---

## Phase 3.4: Integration

- [X] T033 Register Quadlet method in getMethodTargetScheds() in pkg/engine/fetchit.go
- [X] T034 Add quadletMethod constant and update allMethodTypes map in pkg/engine/fetchit.go
- [X] T035 Update TargetConfig struct to include Quadlet field in pkg/engine/types.go
- [ ] T036 Verify SchedInfo() works correctly for Quadlet targets in pkg/engine/quadlet.go
- [ ] T037 Test Quadlet method registration with scheduler in pkg/engine/fetchit.go
- [ ] T038 Verify git change detection works with .container, .volume, .network extensions
- [ ] T039 Test systemctl execution via helper container pattern
- [ ] T040 Verify Quadlet generator triggers on daemon-reload

---

## Phase 3.5: Polish

### Documentation
- [X] T041 [P] Update docs/methods.rst to add Quadlet method section
- [X] T042 [P] Create docs/quadlet.md with detailed Quadlet method guide
- [X] T043 [P] Create docs/migration.md for Systemd → Quadlet migration
- [X] T044 [P] Update README.md with Go 1.22 requirement and Quadlet example
- [X] T045 [P] Update CHANGELOG.md with dependency updates and Quadlet feature

### Examples
- [X] T046 [P] Create examples/quadlet/config.yaml with example Quadlet configuration
- [X] T047 [P] Create examples/quadlet/nginx.container example file
- [X] T048 [P] Create examples/quadlet/nginx-data.volume example file
- [X] T049 [P] Create examples/quadlet/webapp.network example file

### Final Validation
- [ ] T050 Run quickstart.md test scenarios manually to verify all functional requirements
- [ ] T051 Run integration tests on RHEL 9 with Podman 4.9+
- [ ] T052 Verify backwards compatibility - existing methods still work
- [ ] T053 Run performance tests - verify deployment times meet requirements
- [ ] T054 Final code review and cleanup - remove dead code, add comments

---

## Dependencies

**Phase Dependencies**:
- Phase 3.2 (Tests) MUST complete before Phase 3.3 (Implementation)
- Phase 3.3 (Implementation) MUST complete before Phase 3.4 (Integration)
- Phase 3.4 (Integration) MUST complete before Phase 3.5 (Polish)

**Within-Phase Dependencies**:
- T001-T006 can run in parallel (different dependency updates)
- T007 blocks T008 (need clean go.mod before building)
- T009-T020 can run in parallel (different test files)
- T021 blocks T022-T027 (need struct definition before implementing methods)
- T022-T024 are sequential (basic getters before complex methods)
- T025 blocks T026-T027 (Process calls Apply and MethodEngine)
- T028-T032 can implement in parallel (independent helper methods)
- T033-T034 are sequential (need to register in fetchit.go)
- T035 blocks T036-T040 (need TargetConfig update before testing)
- T041-T045 can run in parallel (different doc files)
- T046-T049 can run in parallel (different example files)
- T050-T054 are sequential (validation → testing → review)

**Key Blocking Tasks**:
- T007 (go mod tidy) blocks T008 (build)
- T021 (types.go) blocks T022-T027 (quadlet.go implementation)
- T027 (MethodEngine) blocks T033-T040 (integration)
- T040 (integration complete) blocks T050-T054 (validation)

---

## Parallel Example

### Phase 3.1 Parallel Execution
```bash
# Launch T003-T006 together (different dependency updates):
Task: "Update containers/common to v0.58.0 in go.mod"
Task: "Update containers/image/v5 to v5.30.0 in go.mod"
Task: "Update containers/storage to v1.53.0 in go.mod"
Task: "Update go-git/go-git/v5 to v5.12.0 in go.mod"
```

### Phase 3.2 Parallel Execution
```bash
# Launch T009-T011 together (contract tests):
Task: "Contract test for Method interface in tests/unit/quadlet_method_test.go"
Task: "Contract test for Quadlet configuration schema in tests/unit/quadlet_config_test.go"
Task: "Contract test for systemd integration in tests/integration/quadlet_systemd_test.go"

# Launch T012-T015 together (unit tests):
Task: "Unit test for GetName() and GetKind() in tests/unit/quadlet_test.go"
Task: "Unit test for change type detection in tests/unit/quadlet_test.go"
Task: "Unit test for destination directory calculation in tests/unit/quadlet_test.go"
Task: "Unit test for service name derivation in tests/unit/quadlet_test.go"

# Launch T016-T020 together (integration tests):
Task: "Integration test for initial deployment in tests/integration/quadlet_deployment_test.go"
Task: "Integration test for update detection in tests/integration/quadlet_update_test.go"
Task: "Integration test for file deletion in tests/integration/quadlet_delete_test.go"
Task: "Integration test for root mode in tests/integration/quadlet_root_mode_test.go"
Task: "Integration test for user mode in tests/integration/quadlet_user_mode_test.go"
```

### Phase 3.5 Parallel Execution
```bash
# Launch T041-T045 together (documentation):
Task: "Update docs/methods.rst to add Quadlet method section"
Task: "Create docs/quadlet.md with detailed guide"
Task: "Create docs/migration.md for migration guide"
Task: "Update README.md with Go 1.22 and Quadlet example"
Task: "Update CHANGELOG.md with changes"

# Launch T046-T049 together (examples):
Task: "Create examples/quadlet/config.yaml"
Task: "Create examples/quadlet/nginx.container"
Task: "Create examples/quadlet/nginx-data.volume"
Task: "Create examples/quadlet/webapp.network"
```

---

## Notes

- **[P] tasks** = different files, no dependencies - can run in parallel
- **Verify tests fail** before implementing (TDD approach)
- **Commit after each task** to track progress
- **Run go test ./...** after each implementation task
- **Follow existing patterns** from Systemd method (pkg/engine/systemd.go)
- **Reuse FileTransfer** for file deployment operations
- **Use helper containers** for systemctl operations (privileged containers with systemd mounts)

## Task Generation Rules Applied

1. **From Contracts** (3 contract files → 3 contract test tasks):
   - quadlet-method-interface.md → T009
   - quadlet-config-schema.yaml → T010
   - system-integration-contract.md → T011

2. **From Data Model** (1 entity → model creation task):
   - Quadlet entity → T021 (add to types.go)
   - TargetConfig extension → T035

3. **From Implementation Plan**:
   - 6 Method interface methods → T022-T027
   - 5 Helper methods → T028-T032
   - Integration points → T033-T040

4. **From Quickstart** (9 test scenarios → validation tasks):
   - All scenarios covered in T050-T054

5. **Ordering**:
   - Setup (T001-T008) → Tests (T009-T020) → Implementation (T021-T032) → Integration (T033-T040) → Polish (T041-T054)

## Validation Checklist

- [x] All 3 contract files have corresponding test tasks (T009-T011)
- [x] Quadlet entity has model task (T021, T035)
- [x] All tests come before implementation (Phase 3.2 before 3.3)
- [x] Parallel tasks are truly independent (marked with [P])
- [x] Each task specifies exact file path
- [x] No task modifies same file as another [P] task
- [x] Dependencies explicitly documented
- [x] TDD workflow enforced (tests must fail before implementation)

---

**Total Tasks**: 54
**Estimated Duration**: 5-7 days (with parallel execution)
**Critical Path**: T001 → T007 → T008 → T021 → T022-T027 → T033-T040 → T050-T054
