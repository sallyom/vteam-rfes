# Data Model: FetchIt Modernization

**Feature**: Modernize FetchIt - Dependency Updates and Quadlet Method Support
**Date**: 2025-10-13
**Status**: Phase 1 Design Complete

---

## Overview

This document defines the data entities and structures involved in the FetchIt modernization effort. The modernization introduces a new Quadlet deployment method alongside dependency updates to Go 1.22 and Podman v4.9+. The data model extends the existing FetchIt architecture with minimal changes to ensure backwards compatibility while enabling declarative container management through systemd Quadlet integration.

Key entities include:
- **Quadlet Configuration**: New method configuration for managing Quadlet unit files
- **Quadlet File Types**: `.container`, `.volume`, and `.network` declarative unit files
- **TargetConfig Extension**: Adding Quadlet method to existing configuration structure
- **Dependency Manifest**: Updated go.mod reflecting modern library versions

---

## Key Entities

### Quadlet Configuration

**Entity**: Quadlet (method configuration)

**Purpose**: Configure FetchIt to deploy and manage Podman Quadlet files from a git repository through systemd integration.

**Fields**:
```go
type Quadlet struct {
    CommonMethod `mapstructure:",squash"`

    // Root determines deployment location
    // true: /etc/containers/systemd/ (system-wide, requires root)
    // false: ~/.config/containers/systemd/ (user-level, rootless)
    Root bool `mapstructure:"root"`

    // Enable determines if systemd services should be enabled (auto-start on boot)
    // If true, runs systemctl enable after deployment
    Enable bool `mapstructure:"enable"`

    // Restart determines if services should restart on updates
    // If true, runs systemctl restart when files change
    // Implies Enable=true
    Restart bool `mapstructure:"restart"`
}
```

**Inherited Fields from CommonMethod**:
- `Name` (string): Unique identifier for this Quadlet target within the TargetConfig
- `Schedule` (string): Cron expression for periodic synchronization (e.g., "*/5 * * * *")
- `TargetPath` (string): Directory within git repository containing Quadlet files
- `Glob` (*string): Pattern for filtering specific Quadlet files (e.g., "*.container", "web-*.container")
- `Skew` (*int): Randomization in milliseconds to prevent simultaneous executions
- `initialRun` (bool): Internal flag set by FetchIt to differentiate first execution
- `target` (*Target): Reference to internal Target structure with git URL and auth

**Relationships**:
- **Belongs to**: TargetConfig (one TargetConfig can have multiple Quadlet configurations)
- **References**: Target (git repository URL, branch, authentication)
- **Manages**: Quadlet unit files (.container, .volume, .network) in git repository

**Validation Rules** (from Functional Requirements):

From **FR-011**:
- `Name` must be unique within the TargetConfig's Quadlet array
- `Schedule` must be valid cron expression
- `TargetPath` must be valid relative path within repository
- `Glob` if specified, must be valid glob pattern

From **FR-015 / FR-016**:
- `Root` determines file deployment location:
  - `true`: /etc/containers/systemd/ (requires FetchIt running as root)
  - `false`: ~/.config/containers/systemd/ (requires $HOME environment variable)

From **FR-020**:
- `Schedule` required for automatic synchronization
- Valid cron expressions (e.g., "*/5 * * * *", "0 2 * * *")

From **FR-028**:
- `Glob` supports standard glob patterns: `*`, `?`, `[abc]`, `**`
- Examples: "*.container", "web-*.container", "prod/**/*.container"

**State Transitions**:

```
1. Initial State: Not Deployed
   ├─> Git repository not cloned yet
   └─> initialRun = true

2. Cloning: Git Repository Clone
   ├─> FetchIt clones repository to local volume
   ├─> Identifies Quadlet files matching glob pattern
   └─> Moves to Deployed state

3. Deployed: Files in Systemd Directory
   ├─> Quadlet files copied to /etc/containers/systemd/ or ~/.config/containers/systemd/
   ├─> systemctl daemon-reload executed
   └─> Moves to Service Created state

4. Service Created: Systemd Units Generated
   ├─> Quadlet generator processes .container, .volume, .network files
   ├─> Systemd .service files created in /run/systemd/
   └─> If Enable=true, moves to Running state; else stays in Service Created

5. Running: Containers Active
   ├─> systemctl enable executed (if Enable=true)
   ├─> systemctl start executed (if Enable=true)
   ├─> Containers running and managed by systemd
   └─> Scheduled checks monitor for updates

6. Updated: Configuration Changed
   ├─> Git repository updated with new commit
   ├─> FetchIt detects changes via git diff
   ├─> Files re-deployed, daemon-reload executed
   ├─> If Restart=true, systemctl restart executed
   └─> Returns to Running state

7. Deleted: Files Removed
   ├─> Quadlet file deleted from git repository
   ├─> FetchIt detects deletion via git diff
   ├─> systemctl stop executed for associated service
   ├─> Quadlet file removed from systemd directory
   └─> Returns to Initial State for that specific file
```

**Configuration Example**:
```yaml
targetConfigs:
- name: production-infra
  url: https://github.com/company/infra-configs
  branch: main
  quadlet:
  - name: web-services
    targetPath: production/web/
    glob: "*.container"
    schedule: "*/10 * * * *"
    root: true
    enable: true
    restart: true
  - name: databases
    targetPath: production/databases/
    glob: "*.container"
    schedule: "*/15 * * * *"
    root: true
    enable: true
    restart: false
```

---

### Quadlet File Types

Quadlet files are declarative systemd unit files with Podman-specific syntax. They describe container infrastructure in a format that systemd can process through the Quadlet generator.

#### .container Files

**Purpose**: Define individual container lifecycle, image, ports, volumes, networks, and systemd integration.

**Systemd Translation**: Generates corresponding `.service` unit file in `/run/systemd/system/` or `/run/systemd/user/`

**File Format**: INI-style systemd unit file with Podman-specific sections

**Key Sections**:

1. **[Unit]**: Standard systemd unit metadata
   - Description: Human-readable description
   - After: Service dependencies (e.g., network-online.target)
   - Wants/Requires: Required services

2. **[Container]**: Podman container specification
   - Image: Container image (e.g., docker.io/nginx:latest)
   - PublishPort: Port mappings (e.g., 8080:80)
   - Volume: Volume mounts (e.g., webapp-data:/data:Z)
   - Network: Custom network (references .network file)
   - Environment: Environment variables
   - User/Group: Run-as user/group
   - PodmanArgs: Additional podman arguments

3. **[Service]**: Systemd service configuration
   - Restart: Restart policy (e.g., always, on-failure)
   - TimeoutStartSec: Startup timeout
   - ExecStartPre/ExecStartPost: Pre/post start commands

4. **[Install]**: Systemd installation targets
   - WantedBy: Boot targets (e.g., multi-user.target, default.target)

**Example**:
```ini
[Unit]
Description=Nginx Web Server
After=network-online.target
Wants=network-online.target

[Container]
Image=docker.io/nginx:latest
PublishPort=8080:80
Volume=webapp-data:/usr/share/nginx/html:Z
Network=webapp.network
Environment=NGINX_PORT=80

[Service]
Restart=always
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target default.target
```

**Validation Rules** (from FR-012):
- Must have [Container] section with Image field
- File extension must be `.container`
- Image must be valid container image reference
- PublishPort format: `host_port:container_port[/protocol]`
- Volume format: `name:/path[:options]` or `host_path:/path[:options]`

#### .volume Files

**Purpose**: Define persistent storage volumes with labels, driver options, and lifecycle management.

**Systemd Translation**: Generates `.service` unit that creates named volume via `podman volume create`

**File Format**: INI-style systemd unit file with Volume section

**Key Sections**:

1. **[Unit]**: Standard systemd unit metadata
   - Description: Volume description
   - Before: Ensures volume created before containers

2. **[Volume]**: Podman volume specification
   - Label: Key-value labels for volume metadata
   - Driver: Volume driver (default: local)
   - Options: Driver-specific options

3. **[Install]**: Systemd installation targets
   - WantedBy: When volume should be created

**Example**:
```ini
[Unit]
Description=WebApp Data Volume
Before=webapp.service

[Volume]
Label=app=webapp
Label=environment=production
Driver=local

[Install]
WantedBy=multi-user.target
```

**Validation Rules** (from FR-013):
- File extension must be `.volume`
- Volume name derived from filename (webapp.volume → webapp volume)
- Labels must be key=value format

#### .network Files

**Purpose**: Define custom container networks with subnet, gateway, DNS, and network policies.

**Systemd Translation**: Generates `.service` unit that creates network via `podman network create`

**File Format**: INI-style systemd unit file with Network section

**Key Sections**:

1. **[Unit]**: Standard systemd unit metadata
   - Description: Network description
   - Before: Ensures network created before containers

2. **[Network]**: Podman network specification
   - Subnet: Network CIDR (e.g., 10.88.0.0/16)
   - Gateway: Gateway IP (e.g., 10.88.0.1)
   - IPRange: Allocatable IP range
   - Label: Network labels
   - Driver: Network driver (default: bridge)
   - Options: Driver-specific options
   - DNS: Custom DNS servers

3. **[Install]**: Systemd installation targets
   - WantedBy: When network should be created

**Example**:
```ini
[Unit]
Description=WebApp Private Network
Before=webapp-web.service webapp-db.service

[Network]
Subnet=10.88.0.0/16
Gateway=10.88.0.1
Label=app=webapp
Driver=bridge

[Install]
WantedBy=multi-user.target
```

**Validation Rules** (from FR-014):
- File extension must be `.network`
- Subnet must be valid CIDR notation
- Gateway must be within Subnet range
- Network name derived from filename (webapp.network → webapp network)

---

### TargetConfig Extension

**Existing Structure**: TargetConfig in `/workspace/sessions/agentic-session-1760364724/workspace/fetchit/pkg/engine/types.go`

**Current Fields**:
```go
type TargetConfig struct {
    Name              string             `mapstructure:"name"`
    Url               string             `mapstructure:"url"`
    Device            string             `mapstructure:"device"`
    Disconnected      bool               `mapstructure:"disconnected"`
    VerifyCommitsInfo *VerifyCommitsInfo `mapstructure:"verifyCommitsInfo"`
    Branch            string             `mapstructure:"branch"`
    Ansible           []*Ansible         `mapstructure:"ansible"`
    FileTransfer      []*FileTransfer    `mapstructure:"filetransfer"`
    Kube              []*Kube            `mapstructure:"kube"`
    Raw               []*Raw             `mapstructure:"raw"`
    Systemd           []*Systemd         `mapstructure:"systemd"`

    // Internal fields
    image        *Image
    prune        *Prune
    configReload *ConfigReload
    mu           sync.Mutex
}
```

**New Field to Add**:
```go
Quadlet []*Quadlet `mapstructure:"quadlet"`
```

**Impact on Existing Code**:

1. **Configuration Parsing** (config.go):
   - Viper will automatically unmarshal `quadlet:` section from YAML
   - No changes needed to parsing logic (mapstructure handles it)

2. **Method Registration** (fetchit.go, getMethodTargetScheds function):
   - Add new block to register Quadlet methods:
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

3. **Backwards Compatibility**:
   - Existing configurations without `quadlet:` section continue to work
   - No breaking changes to other method configurations
   - Optional field, defaults to empty array if not specified

**Example Extended Configuration**:
```yaml
targetConfigs:
- name: production-services
  url: https://github.com/company/service-configs
  branch: main
  # Existing methods continue to work
  systemd:
  - name: legacy-service
    targetPath: systemd/
    schedule: "*/5 * * * *"
    root: true
  # New Quadlet method
  quadlet:
  - name: modern-containers
    targetPath: quadlet/
    glob: "*.container"
    schedule: "*/10 * * * *"
    root: true
    enable: true
    restart: true
```

---

### Dependency Manifest

**Entity**: go.mod file

**Purpose**: Declare all direct and transitive dependencies with version constraints for the FetchIt project.

**Current State** (as of FetchIt codebase inspection):
```go
module github.com/containers/fetchit

go 1.17

require (
    github.com/containers/podman/v4 v4.2.0
    github.com/containers/common v0.49.1
    github.com/containers/image/v5 v5.22.1
    github.com/containers/storage v1.42.1
    github.com/go-git/go-git/v5 v5.11.0
    github.com/go-co-op/gocron v1.x.x
    github.com/spf13/viper v1.13.0
    github.com/spf13/cobra v1.5.0
    // ... additional dependencies
)
```

**Target State** (after modernization):
```go
module github.com/containers/fetchit

go 1.22

require (
    github.com/containers/podman/v4 v4.9.4
    github.com/containers/common v0.58.0
    github.com/containers/image/v5 v5.30.0
    github.com/containers/storage v1.53.0
    github.com/go-git/go-git/v5 v5.12.0
    github.com/go-co-op/gocron v1.x.x
    github.com/spf13/viper v1.18.0
    github.com/spf13/cobra v1.8.0
    github.com/opencontainers/runtime-spec v1.1.0
    go.uber.org/zap v1.27.0
    // ... additional dependencies with updated versions
)
```

**go.sum Changes**:
- Transitive dependency hashes updated for all direct dependency updates
- New transitive dependencies from Podman v4.9
- Removed obsolete transitive dependencies from Podman v4.2

**Validation Requirements** (from FR-005, FR-006):

1. **CVE Scanning**:
   - Run `govulncheck ./...` to identify vulnerabilities
   - No high or critical severity CVEs should remain
   - Medium CVEs documented with risk acceptance if applicable

2. **Build Verification**:
   - `go mod tidy` must succeed without errors
   - `go build ./...` must succeed on AMD64 and ARM64
   - All existing tests must pass after updates

3. **Consistency Checks**:
   - go.mod and go.sum must be in sync
   - No replace directives for production dependencies
   - Version constraints must allow compatible updates

**Update Process** (from research.md):
1. Update Go version: `go 1.22`
2. Update Podman libraries: `go get github.com/containers/podman/v4@v4.9.4`
3. Update containers ecosystem: `go get github.com/containers/common@v0.58.0`
4. Update go-git: `go get github.com/go-git/go-git/v5@v5.12.0`
5. Update transitive dependencies: `go get -u ./...` (carefully)
6. Tidy: `go mod tidy`
7. Vendor (if needed): `go mod vendor`

---

## State Machine

This section documents the complete lifecycle of a Quadlet deployment from initial configuration through updates and eventual deletion.

### Primary State Flow

```
[Not Configured]
      ↓ (User adds Quadlet target to config.yaml)
      ↓
[Configured - Initial Run]
      ↓ (FetchIt starts, initialRun=true)
      ↓
[Cloning Repository]
      ↓ (git clone executes)
      ↓
[Repository Cloned] ←──────┐
      ↓                     │ (Scheduled check, changes detected)
      ↓                     │
[Detecting Changes]         │
      ↓                     │
      ├─> No changes ───────┘ (currentToLatest returns early)
      │
      ├─> Changes detected
      ↓
[Processing Changes]
      ↓
      ├─> Create: New Quadlet file added
      │   ├─> Deploy file to systemd directory
      │   ├─> systemctl daemon-reload
      │   └─> If Enable=true: systemctl enable + start
      │
      ├─> Update: Existing Quadlet file modified
      │   ├─> Update file in systemd directory
      │   ├─> systemctl daemon-reload
      │   └─> If Restart=true: systemctl restart
      │
      ├─> Delete: Quadlet file removed
      │   ├─> systemctl stop service
      │   └─> Remove file from systemd directory
      │
      └─> Rename: Quadlet file renamed
          ├─> systemctl stop old-service
          ├─> Remove old file
          ├─> Deploy new file
          ├─> systemctl daemon-reload
          └─> If Enable=true: systemctl enable + start new-service
      ↓
[Changes Applied]
      ↓ (Update git tag: current-quadlet-{name})
      ↓
[Deployed] ────────────────────┘ (Wait for next scheduled check)
```

### Detailed State Descriptions

**1. Not Configured**
- Initial state before Quadlet target added to config.yaml
- No Quadlet method registered in FetchIt
- No scheduled jobs exist for Quadlet

**2. Configured - Initial Run**
- User adds Quadlet target to config.yaml
- FetchIt reads configuration via Viper
- Quadlet method instantiated with `initialRun=true`
- Scheduled job registered with gocron

**3. Cloning Repository**
- `getRepo(target)` executes
- Git repository cloned to FetchIt volume
- Clone location: `/opt/volume/{repo-name}/`
- Authentication via SSH key, PAT, or username/password

**4. Repository Cloned**
- Git repository present on filesystem
- Ready for change detection
- Git tag `current-quadlet-{name}` may not exist yet

**5. Detecting Changes**
- `zeroToCurrent()` on initial run: Deploys files from current commit
- `currentToLatest()` on subsequent runs: Compares current tag to latest commit
- Git diff executed to identify file changes
- Glob filter applied to match Quadlet files

**6. Processing Changes**
- For each detected change, `MethodEngine()` invoked
- Change type determined: create, update, delete, rename
- Files filtered by extensions: .container, .volume, .network

**7. Changes Applied**
- All changes processed successfully
- Git tag updated: `current-quadlet-{name} → new-commit-hash`
- Systemd services running (if Enable=true)

**8. Deployed**
- Steady state between scheduled checks
- Containers running as systemd services
- Next scheduled check will detect any new changes

### Error State Transitions

```
[Any State]
      ↓ (Error occurs)
      ↓
[Error State]
      ↓
      ├─> Git clone failure
      │   └─> Retry on next scheduled check
      │
      ├─> Systemd not available
      │   └─> Log error, skip processing
      │
      ├─> Permission denied
      │   └─> Log error, require user intervention
      │
      ├─> Malformed Quadlet file
      │   └─> Log error, skip file, continue with others
      │
      └─> Podman version too old
          └─> Log error, require upgrade
      ↓
[Wait for Next Schedule] ──> [Retry or User Fix]
```

### Concurrency Control

- Mutex (`target.mu`) prevents concurrent operations on same target
- Only one `Process()` execution per target at a time
- Scheduled checks skipped if previous run still in progress

---

## Validation Rules

This section consolidates validation rules from the functional requirements (FR-010 through FR-029).

### Configuration Validation (FR-011, FR-020)

**Quadlet Method Configuration**:
- `name`: Required, must be unique within TargetConfig.Quadlet array, alphanumeric + hyphens/underscores
- `targetPath`: Required, must be valid relative path, no absolute paths or ".." references
- `schedule`: Required, must be valid cron expression (5 or 6 fields)
- `glob`: Optional, if specified must be valid glob pattern
- `root`: Required boolean (default: false)
- `enable`: Required boolean (default: false)
- `restart`: Required boolean (default: false)

**Schedule Validation Examples**:
- Valid: `"*/5 * * * *"` (every 5 minutes)
- Valid: `"0 2 * * *"` (daily at 2 AM)
- Valid: `"0 0 * * 0"` (weekly on Sunday)
- Invalid: `"invalid"` (not cron format)
- Invalid: `"* * * *"` (too few fields)

**Glob Pattern Validation**:
- Valid: `"*.container"` (all .container files)
- Valid: `"web-*.container"` (web-prefixed .container files)
- Valid: `"prod/**/*.container"` (recursive .container files in prod/)
- Invalid: `"[*.container"` (unmatched bracket)

### Runtime Environment Validation (FR-027)

**Podman Version Check**:
```
Minimum: 4.4.0
Recommended: 4.9.4
Command: podman --version
Expected: Podman version 4.4.0 or higher
Error if: Version < 4.4.0
```

**cgroup v2 Check**:
```
Command: podman info --format '{{.Host.CgroupsVersion}}'
Expected: v2
Error if: v1 or empty
Message: "Quadlet requires cgroup v2. See https://docs.podman.io/en/latest/markdown/podman-system-migrate.1.html"
```

**systemd Check**:
```
Command: systemctl --version
Expected: Systemd version number
Error if: Command not found or fails
Message: "Quadlet requires systemd. Consider using Raw or Kube methods instead."
```

**Directory Writability Check**:
```
Root mode:
  Path: /etc/containers/systemd/
  Check: Directory exists and writable
  Create: If missing, create with 0755 permissions

User mode:
  Path: ~/.config/containers/systemd/
  Check: $HOME environment variable set, directory exists and writable
  Create: If missing, create with 0755 permissions
  Error if: $HOME not set
```

### Quadlet File Validation (FR-012, FR-013, FR-014)

**.container File Validation**:
- Must have [Container] section
- [Container] section must have `Image=` directive
- Image format: `[registry/]name[:tag|@digest]`
- PublishPort format: `host_port:container_port[/tcp|udp]`
- Volume format: `name:/path[:Z|z|ro|rw]` or `/host/path:/path[:Z|z|ro|rw]`
- Network reference: Must match existing .network filename (without extension)

**.volume File Validation**:
- Must have [Volume] section
- Volume name derived from filename (no validation of name format)
- Label format: `key=value` (one per Label= line)
- Driver must be valid volume driver (typically `local`)

**.network File Validation**:
- Must have [Network] section
- Subnet must be valid CIDR: IPv4 `x.x.x.x/y` or IPv6 `xxxx::/y`
- Gateway must be IP within Subnet range
- Label format: `key=value` (one per Label= line)
- Driver must be valid network driver (typically `bridge`)

### Change Detection Validation (FR-018, FR-019)

**Git Change Detection**:
- Compare `current-quadlet-{name}` tag to latest commit hash
- If different, compute git diff between commits
- Filter changes by targetPath
- Filter changes by glob pattern
- Filter changes by file extensions: [".container", ".volume", ".network"]

**Change Type Determination**:
```
Create:  change.From.Name == "" && change.To.Name != ""
Update:  change.From.Name == change.To.Name
Rename:  change.From.Name != change.To.Name && both non-empty
Delete:  change.From.Name != "" && change.To.Name == ""
```

### Deployment Validation (FR-015, FR-016, FR-017, FR-021)

**File Deployment**:
- Copy Quadlet file from git working tree to systemd directory
- Preserve file permissions
- File must be readable by systemd (0644 recommended)
- Filename must have correct extension (.container, .volume, .network)

**systemd Integration**:
- Execute `systemctl daemon-reload` (root) or `systemctl --user daemon-reload` (user)
- Verify daemon-reload exit code 0
- If Enable=true: Execute `systemctl enable {service}` or `systemctl --user enable {service}`
- If Restart=true: Execute `systemctl restart {service}` or `systemctl --user restart {service}`

**Success Verification** (FR-022, FR-023, FR-029):
- For .container files: Verify container running via `podman ps`
- For systemd services: Verify service active via `systemctl status {service}`
- Log deployment success with commit hash and file names

### Error Handling and Logging (FR-025, FR-026)

**Malformed Quadlet Files**:
- Detect missing required sections ([Container], [Volume], [Network])
- Detect missing required directives (Image= in [Container])
- Log clear error: "Quadlet file {filename} is invalid: missing required section/directive"
- Skip invalid file, continue processing other files
- Do not fail entire deployment due to one invalid file

**Systemd Errors**:
- Capture stderr from systemctl commands
- Log full error output
- Include file path and command that failed
- Provide troubleshooting hints (check file syntax, check systemd logs)

**Deployment Logging**:
- Log level INFO: Successful deployments, file counts, commit hashes
- Log level DEBUG: Git operations, file filtering, change detection
- Log level ERROR: Failures, validation errors, systemd errors
- Include: Quadlet target name, git URL, commit hash, affected files

**Log Message Examples**:
```
INFO: Quadlet target 'web-services' deployed 3 files from commit abc1234
DEBUG: Filtered 15 files to 3 matching glob '*.container' in path 'production/web/'
ERROR: Quadlet file 'invalid.container' missing [Container] section, skipping
ERROR: systemctl enable webapp.service failed: Unit file not found
```

---

## Summary

This data model provides a comprehensive specification for extending FetchIt with Quadlet support while maintaining backwards compatibility. Key design decisions:

1. **Minimal Changes**: Only adds `Quadlet` field to `TargetConfig`, reuses existing infrastructure
2. **Consistency**: Follows established patterns from Raw, Systemd, Kube, Ansible, FileTransfer methods
3. **Validation**: Comprehensive validation at configuration, runtime environment, and file format levels
4. **State Management**: Clear state machine with well-defined transitions and error handling
5. **Backwards Compatibility**: Existing configurations continue to work without modification

The design enables declarative container management through systemd while leveraging FetchIt's proven git-based synchronization and scheduling capabilities.
