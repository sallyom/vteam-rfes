# Feature Specification: Modernize FetchIt - Dependency Updates and Quadlet Method Support

**Feature Branch**: `001-we-need-to`
**Created**: 2025-10-13
**Status**: Draft
**Input**: User description: "We need to update fetchit project's dependencies. Update the podman version that's pulled in. Add ability to configure quadlets, too."

## Execution Flow (main)
```
1. Parse user description from Input
   → Identified: dependency updates (Go, Podman libraries) and new Quadlet method
2. Extract key concepts from description
   → Actors: FetchIt operators, system administrators, DevOps engineers
   → Actions: update dependencies, deploy Quadlet files, manage container lifecycle
   → Data: Quadlet unit files (.container, .volume, .network), configuration YAML
   → Constraints: backwards compatibility, systemd integration, Podman 4.4+ requirement
3. For each unclear aspect:
   → [CLARIFIED VIA RFE.MD]: Target Go version (1.21+), Podman v4.9+ or v5.x, Quadlet file types
4. Fill User Scenarios & Testing section
   → Primary flow: User deploys containers via Quadlet method through git-tracked files
5. Generate Functional Requirements
   → All requirements testable via automated tests and manual verification
6. Identify Key Entities
   → Quadlet files, TargetConfig, systemd units
7. Run Review Checklist
   → Spec focuses on WHAT/WHY, not HOW (implementation in planning phase)
8. Return: SUCCESS (spec ready for planning)
```

---

## ⚡ Quick Guidelines
- ✅ Focus on WHAT users need and WHY
- ❌ Avoid HOW to implement (no tech stack, APIs, code structure)
- 👥 Written for business stakeholders, not developers

---

## User Scenarios & Testing

### Primary User Story

As a **system administrator managing edge infrastructure**, I want to deploy and update Podman containers using declarative Quadlet files tracked in git, so that I can leverage modern Podman tooling for systemd integration while ensuring my FetchIt deployment uses secure, up-to-date dependencies.

**Why this matters:**
- Current FetchIt dependencies (Go 1.17, older Podman libraries) contain security vulnerabilities and lack modern features
- Podman has deprecated `podman generate systemd` in favor of Quadlets (introduced in 4.4+)
- Managing systemd units manually is complex and error-prone
- GitOps workflows require reliable, declarative configuration approaches

### Acceptance Scenarios

1. **Dependency Update Success**
   - **Given** FetchIt is currently using Go 1.17 and Podman v4 libraries with known CVEs
   - **When** dependencies are updated to Go 1.21+ and Podman v4.9+/v5.x
   - **Then** all existing methods (Raw, Systemd, Kube, Ansible, FileTransfer) continue to function without regression
   - **And** no high or critical severity CVEs remain in the dependency tree
   - **And** FetchIt builds successfully on AMD64 and ARM64 architectures

2. **Quadlet Deployment for Root User**
   - **Given** a git repository containing webapp.container, webapp.volume, and webapp.network Quadlet files
   - **When** user configures FetchIt with a Quadlet target pointing to this repository with `root: true`
   - **Then** FetchIt deploys the Quadlet files to `/etc/containers/systemd/`
   - **And** systemd daemon reloads to register the new units
   - **And** the container starts as a systemd service
   - **And** the container persists across system reboots

3. **Quadlet Deployment for Non-Root User**
   - **Given** a git repository containing Quadlet files for a user application
   - **When** user configures FetchIt with a Quadlet target with `root: false`
   - **Then** FetchIt deploys the Quadlet files to `~/.config/containers/systemd/`
   - **And** services run in the user systemd context
   - **And** user can manage services with `systemctl --user` commands

4. **Automatic Updates on Git Changes**
   - **Given** a container is running via Quadlet method managed by FetchIt
   - **When** the user updates the .container file in the git repository (e.g., changes image tag)
   - **Then** FetchIt detects the change on the next scheduled check
   - **And** FetchIt updates the systemd Quadlet file
   - **And** systemd restarts the container with the new configuration
   - **And** the container runs successfully with updated settings

5. **Multi-Container Application with Dependencies**
   - **Given** an application requiring a database container, web container, shared volume, and custom network
   - **When** user commits myapp.volume, myapp-db.container, myapp-web.container, and myapp.network files to git
   - **Then** FetchIt deploys all Quadlet files to the systemd directory
   - **And** systemd starts services in the correct dependency order
   - **And** containers can communicate over the custom network
   - **And** data persists in the shared volume

6. **Scheduled Quadlet Synchronization**
   - **Given** a Quadlet target configured with schedule `*/5 * * * *` (every 5 minutes)
   - **When** the schedule triggers
   - **Then** FetchIt pulls the latest changes from the git repository
   - **And** FetchIt processes any new or modified Quadlet files
   - **And** FetchIt applies changes to systemd
   - **And** affected services restart if needed

### Edge Cases

- **What happens when a Quadlet file references a non-existent volume or network?**
  - FetchIt should create those, but with very clear logs. 

- **How does the system handle malformed Quadlet files?**
  - System must detect syntax errors and report them clearly to the user without disrupting other running containers

- **What happens when multiple Quadlet files change simultaneously in a single git commit?**
  - System must process all changes atomically and handle systemd reloads efficiently to avoid race conditions

- **How does the system handle Quadlet deployment when systemd is not available?**
  - System must detect the absence of systemd and fail gracefully with an informative error message

- **What happens when a dependency update introduces breaking API changes?**
  - Existing functionality must continue to work, or upgrade path documentation must be provided

- **How are Quadlet file naming conflicts handled?**
  - System must detect conflicts (e.g., two repos deploying the same .container filename) and report the issue clearly

## Requirements

### Functional Requirements

**Dependency Updates:**

- **FR-001**: System MUST update Go version to 1.21 or later to benefit from security patches, performance improvements, and modern language features
- **FR-002**: System MUST update `github.com/containers/podman/v4` to v4.9+ or v5.x (if stable) to access modern Podman capabilities and security fixes
- **FR-003**: System MUST update `github.com/containers/common` to the latest compatible version
- **FR-004**: System MUST update `github.com/go-git/go-git/v5` to address known security vulnerabilities
- **FR-005**: System MUST update all transitive dependencies with known high or critical severity CVEs
- **FR-006**: System MUST maintain `go.mod` and `go.sum` in a consistent, properly updated state
- **FR-007**: System MUST build successfully on both AMD64 and ARM64 architectures after dependency updates
- **FR-008**: System MUST preserve backwards compatibility for all existing deployment methods (Raw, Systemd, Kube, Ansible, FileTransfer) after dependency updates
- **FR-009**: Existing example configurations in the examples/ directory MUST continue to work without modification after dependency updates

**Quadlet Method Implementation:**

- **FR-010**: System MUST support a new "Quadlet" deployment method as a first-class target type alongside existing methods
- **FR-011**: Users MUST be able to define Quadlet targets in config.yaml with name, targetPath, schedule, root flag, and glob patterns
- **FR-012**: System MUST detect and process .container Quadlet files for container deployment
- **FR-013**: System MUST detect and process .volume Quadlet files for persistent storage configuration
- **FR-014**: System MUST detect and process .network Quadlet files for custom networking configuration
- **FR-015**: System MUST deploy Quadlet files to `/etc/containers/systemd/` when `root: true` is specified
- **FR-016**: System MUST deploy Quadlet files to `~/.config/containers/systemd/` when `root: false` is specified
- **FR-017**: System MUST trigger `systemctl daemon-reload` after deploying or updating Quadlet files to register units with systemd
- **FR-018**: System MUST detect changes to Quadlet files in git repositories on scheduled checks
- **FR-019**: System MUST update deployed Quadlet files and restart affected services when git changes are detected
- **FR-020**: System MUST support cron expression scheduling for Quadlet targets to enable automatic synchronization
- **FR-021**: System MUST integrate Quadlet method with existing git change detection and apply logic
- **FR-022**: Containers deployed via Quadlet MUST be visible via `podman ps` command
- **FR-023**: Containers deployed via Quadlet MUST be manageable via `systemctl status` and related systemctl commands
- **FR-024**: Containers deployed via Quadlet MUST persist across system reboots when configured to do so

**Configuration and User Experience:**

- **FR-025**: System MUST provide clear error messages when Quadlet files are malformed or invalid
- **FR-026**: System MUST log Quadlet deployment activities for troubleshooting and auditing
- **FR-027**: System MUST validate that Quadlet method is only used on systems with Podman 4.4+ and systemd support
- **FR-028**: System MUST support glob patterns for filtering Quadlet files (e.g., `*.container`, `web-*.container`)
- **FR-029**: Users MUST be able to verify Quadlet deployment success via standard systemd and Podman tooling

**Documentation:**

- **FR-030**: Documentation MUST be updated to reflect the new Go version requirement
- **FR-031**: Documentation MUST include comprehensive Quadlet method configuration examples
- **FR-032**: Documentation MUST provide example Quadlet files for .container, .volume, and .network types
- **FR-033**: Documentation MUST explain the differences between Raw, Systemd, and Quadlet methods to help users choose appropriately
- **FR-034**: Documentation MUST provide migration guidance for users transitioning from Systemd to Quadlet method
- **FR-035**: Release notes or CHANGELOG MUST document dependency updates and list any breaking changes

### Key Entities

- **Quadlet File**: A declarative systemd unit file with Podman-specific syntax (extensions: .container, .volume, .network, .kube, .pod, .build, .image) that describes container lifecycle, networking, storage, or related resources. These files are the primary configuration artifact for the Quadlet method.

- **TargetConfig (Quadlet Type)**: Configuration structure in config.yaml that defines how FetchIt should handle a Quadlet-based git repository. Includes attributes such as:
  - `name`: Identifier for the target
  - `url`: Git repository URL
  - `branch`: Git branch to track
  - `targetPath`: Directory within the repository containing Quadlet files
  - `root`: Boolean flag indicating whether to deploy as root (system-wide) or user-level
  - `schedule`: Cron expression for periodic synchronization
  - `glob`: Pattern for filtering specific Quadlet files

- **Systemd Unit**: The runtime representation of a deployed Quadlet file, managed by systemd. FetchIt interacts with systemd to reload daemon configuration and manage service lifecycle.

- **Dependency Manifest**: The `go.mod` file that declares all direct and transitive dependencies with version constraints. Must be updated to reflect modern library versions while maintaining compatibility.

---

## Review & Acceptance Checklist

### Content Quality
- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs (security, modern tooling, simplified management)
- [x] Written for non-technical stakeholders (clear use cases and scenarios)
- [x] All mandatory sections completed

### Requirement Completeness
- [x] No [NEEDS CLARIFICATION] markers remain (1 edge case needs clarification on volume/network handling)
- [x] Requirements are testable and unambiguous (all FRs have clear verification criteria)
- [x] Success criteria are measurable (defined in acceptance scenarios and RFE)
- [x] Scope is clearly bounded (MVP focuses on .container, .volume, .network; excludes .kube, .pod, .build)
- [x] Dependencies and assumptions identified (Podman 4.4+, systemd availability, Go 1.21+)

---

## Execution Status

- [x] User description parsed
- [x] Key concepts extracted (dependency updates, Quadlet method, systemd integration)
- [x] Ambiguities marked (1 clarification needed on edge case handling)
- [x] User scenarios defined (6 acceptance scenarios covering main flows)
- [x] Requirements generated (35 functional requirements, all testable)
- [x] Entities identified (Quadlet files, TargetConfig, systemd units, dependency manifest)
- [x] Review checklist passed (pending edge case clarification)

---

## Next Steps

1. **Planning Phase**: Once clarification is complete, proceed to `/plan` to generate design artifacts and implementation plan
2. **Task Breakdown**: After planning, use `/tasks` to create dependency-ordered implementation tasks
3. **Implementation**: Execute tasks via `/implement` workflow

---

## References

- **Original RFE**: `/workspace/sessions/agentic-session-1760359514/workspace/vteam-rfes/rfe.md`
- **FetchIt Repository**: https://github.com/containers/fetchit
- **Podman Quadlet Documentation**: https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html
- **Existing Methods Documentation**: https://github.com/containers/fetchit/blob/main/docs/methods.rst
