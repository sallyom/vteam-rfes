# Method Interface Contract: Quadlet

**Feature**: FetchIt Modernization - Quadlet Method Implementation
**Date**: 2025-10-13
**Status**: Design Complete

---

## Overview

This document specifies the contract for implementing the Quadlet deployment method in FetchIt. The Quadlet method must implement the `Method` interface defined in `/workspace/sessions/agentic-session-1760364724/workspace/fetchit/pkg/engine/types.go` to integrate with FetchIt's scheduling, git change detection, and deployment infrastructure.

The Quadlet method follows the same architectural pattern as existing methods (Raw, Systemd, Kube, Ansible, FileTransfer) to maintain consistency and leverage proven code patterns.

---

## Method Interface Definition

The `Method` interface requires six methods to be implemented:

```go
type Method interface {
    GetName() string
    GetKind() string
    GetTarget() *Target
    Process(ctx context.Context, conn context.Context, skew int)
    Apply(ctx context.Context, conn context.Context, currentState plumbing.Hash, desiredState plumbing.Hash, tags *[]string) error
    MethodEngine(ctx context.Context, conn context.Context, change *object.Change, path string) error
}
```

---

## Interface Method Contracts

### GetName() string

**Purpose**: Returns the unique identifier for this specific Quadlet target instance.

**Contract**:
- **Input**: None
- **Output**: String containing the `Name` field from the Quadlet configuration
- **Behavior**: Returns the value set in config.yaml under `quadlet[].name`
- **Invariants**:
  - Must be unique within the parent TargetConfig's Quadlet array
  - Must not change during the lifetime of the Quadlet instance
  - Used for git tag naming: `current-quadlet-{name}`

**Implementation**:
```go
func (q *Quadlet) GetName() string {
    return q.Name  // Inherited from CommonMethod
}
```

**Example**:
```yaml
quadlet:
- name: web-services  # GetName() returns "web-services"
```

**Usage in FetchIt**:
- Logging: "Processing Quadlet target: {GetName()}"
- Git tags: "current-quadlet-{GetName()}"
- Scheduling: Identifies which method instance is executing

---

### GetKind() string

**Purpose**: Returns the method type identifier for Quadlet.

**Contract**:
- **Input**: None
- **Output**: String constant "quadlet"
- **Behavior**: Always returns the string literal "quadlet"
- **Invariants**:
  - Must return exactly "quadlet" (lowercase)
  - Used for method type tagging in scheduler
  - Used for git tag prefixes

**Implementation**:
```go
const quadletMethod = "quadlet"

func (q *Quadlet) GetKind() string {
    return quadletMethod
}
```

**Usage in FetchIt**:
- Scheduler tags: Groups all Quadlet jobs for batch operations
- Git tags: "current-quadlet-{name}" format
- Logging: "Method: quadlet"
- Method type tracking: `fetchit.allMethodTypes["quadlet"]`

---

### GetTarget() *Target

**Purpose**: Returns the internal Target structure containing git repository URL, authentication, and branch information.

**Contract**:
- **Input**: None
- **Output**: Pointer to Target struct
- **Behavior**: Returns the `target` field set during initialization
- **Invariants**:
  - Must not return nil after initialization
  - Target contains: url, branch, ssh/pat/username/password, device, localPath

**Implementation**:
```go
func (q *Quadlet) GetTarget() *Target {
    return q.target  // Inherited from CommonMethod
}
```

**Target Structure**:
```go
type Target struct {
    ssh             bool
    sshKey          string
    url             string
    pat             string
    envSecret       string
    username        string
    password        string
    device          string
    localPath       string
    branch          string
    mu              sync.Mutex
    disconnected    bool
    gitsignVerify   bool
    gitsignRekorURL string
}
```

**Usage in FetchIt**:
- Git operations: Clone, fetch, checkout
- Mutex locking: Prevent concurrent operations on same target
- Logging: Include repository URL in messages

---

### Process(ctx context.Context, conn context.Context, skew int)

**Purpose**: Main entry point called by the scheduler on each scheduled run. Orchestrates the complete Quadlet deployment workflow.

**Contract**:
- **Input**:
  - `ctx`: Context for git operations and cancellation
  - `conn`: Podman connection context for container/systemd operations
  - `skew`: Random delay in milliseconds to prevent thundering herd
- **Output**: None (logs errors, does not return error)
- **Behavior**:
  1. Sleep for `skew` milliseconds
  2. Acquire target mutex lock
  3. If `initialRun == true`:
     - Call `getRepo(target)` to clone git repository
     - Call `zeroToCurrent()` to deploy files from current commit
     - Set `initialRun = false`
  4. If not initial run:
     - Call `currentToLatest()` to check for changes and apply updates
  5. Release target mutex lock
- **Error Handling**:
  - Log errors but do not return them (scheduler expects void)
  - Failed runs will retry on next scheduled execution
  - Partial deployments logged with file counts

**Implementation Pattern** (following Systemd method):
```go
func (q *Quadlet) Process(ctx, conn context.Context, skew int) {
    target := q.GetTarget()
    time.Sleep(time.Duration(skew) * time.Millisecond)
    target.mu.Lock()
    defer target.mu.Unlock()

    tag := []string{".container", ".volume", ".network"}

    if q.Restart {
        q.Enable = true  // Restart implies Enable
    }

    if q.initialRun {
        err := getRepo(target)
        if err != nil {
            logger.Errorf("Failed to clone repository %s: %v", target.url, err)
            return
        }

        err = zeroToCurrent(ctx, conn, q, target, &tag)
        if err != nil {
            logger.Errorf("Error moving to current: %v", err)
            return
        }
    }

    err := currentToLatest(ctx, conn, q, target, &tag)
    if err != nil {
        logger.Errorf("Error moving current to latest: %v", err)
        return
    }

    q.initialRun = false
}
```

**Scheduling Context**:
- Called by gocron scheduler based on `Schedule` cron expression
- May be called concurrently for different Quadlet targets
- Mutex ensures same target not processed concurrently

---

### Apply(ctx context.Context, conn context.Context, currentState plumbing.Hash, desiredState plumbing.Hash, tags *[]string) error

**Purpose**: Computes the difference between two git commits and applies the changes by deploying, updating, or removing Quadlet files.

**Contract**:
- **Input**:
  - `ctx`: Context for git operations
  - `conn`: Podman connection context
  - `currentState`: Git commit hash representing current deployed state (may be ZeroHash on initial run)
  - `desiredState`: Git commit hash representing desired state (latest commit or current tag)
  - `tags`: Pointer to array of file extensions to filter: `[".container", ".volume", ".network"]`
- **Output**: Error if deployment fails, nil on success
- **Behavior**:
  1. Call `applyChanges()` to compute diff between currentState and desiredState
  2. Filter changes by `TargetPath`, `Glob`, and `tags`
  3. Build map of `*object.Change` → `string` (file path)
  4. Call `runChanges()` to iterate over change map and invoke `MethodEngine()` for each
  5. Return error if any deployment fails
- **Error Handling**:
  - Return first error encountered
  - Do not continue if one file fails (fail-fast)
  - Caller will retry on next scheduled run

**Implementation Pattern** (following Systemd method):
```go
func (q *Quadlet) Apply(ctx, conn context.Context, currentState, desiredState plumbing.Hash, tags *[]string) error {
    changeMap, err := applyChanges(ctx, q.GetTarget(), q.GetTargetPath(), q.Glob, currentState, desiredState, tags)
    if err != nil {
        return err
    }
    if err := runChanges(ctx, conn, q, changeMap); err != nil {
        return err
    }
    return nil
}
```

**Change Map Structure**:
```go
map[*object.Change]string{
    change1: "/path/to/repo/quadlet/webapp.container",
    change2: "/path/to/repo/quadlet/database.container",
    change3: "/path/to/repo/quadlet/app.network",
}
```

**Called By**:
- `zeroToCurrent()`: Deploy files from current tagged commit on initial run
- `currentToLatest()`: Deploy changes when moving from current to latest commit

---

### MethodEngine(ctx context.Context, conn context.Context, change *object.Change, path string) error

**Purpose**: Handles the deployment of a single Quadlet file change. This is the method-specific logic that distinguishes Quadlet from other deployment methods.

**Contract**:
- **Input**:
  - `ctx`: Context for operations
  - `conn`: Podman connection context for systemctl operations
  - `change`: Git change object containing From/To file names and modes (may be nil for initial deploy)
  - `path`: Absolute path to the Quadlet file in the git working tree
- **Output**: Error if deployment fails, nil on success
- **Behavior**:
  1. Determine change type:
     - **Create**: `change.From.Name == ""` && `change.To.Name != ""`
     - **Update**: `change.From.Name == change.To.Name`
     - **Rename**: `change.From.Name != change.To.Name` && both non-empty
     - **Delete**: `change.From.Name != ""` && `change.To.Name == ""`
  2. Determine destination directory:
     - If `Root == true`: `/etc/containers/systemd/`
     - If `Root == false`: `$HOME/.config/containers/systemd/`
  3. Execute appropriate actions based on change type:
     - **Create**: Deploy file → daemon-reload → enable (if Enable=true)
     - **Update**: Update file → daemon-reload → restart (if Restart=true)
     - **Rename**: Stop old service → remove old file → deploy new file → daemon-reload → enable new service
     - **Delete**: Stop service → remove file
  4. Return error if any step fails

**Implementation Pattern** (following Systemd method):
```go
func (q *Quadlet) MethodEngine(ctx context.Context, conn context.Context, change *object.Change, path string) error {
    var changeType string = "unknown"
    var curr *string = nil
    var prev *string = nil

    if change != nil {
        if change.From.Name != "" {
            prev = &change.From.Name
        }
        if change.To.Name != "" {
            curr = &change.To.Name
        }
        if change.From.Name == "" && change.To.Name != "" {
            changeType = "create"
        }
        if change.From.Name != "" && change.To.Name != "" {
            if change.From.Name == change.To.Name {
                changeType = "update"
            } else {
                changeType = "rename"
            }
        }
        if change.From.Name != "" && change.To.Name == "" {
            changeType = "delete"
        }
    }

    nonRootHomeDir := os.Getenv("HOME")
    if nonRootHomeDir == "" {
        return fmt.Errorf("Could not determine $HOME for host, must set $HOME on host machine")
    }

    var dest string
    if q.Root {
        dest = "/etc/containers/systemd"
    } else {
        dest = filepath.Join(nonRootHomeDir, ".config", "containers", "systemd")
    }

    if change != nil {
        q.initialRun = true  // FileTransfer needs this flag
    }

    return q.quadletPodman(ctx, conn, path, dest, prev, curr, &changeType)
}
```

**Key Operations**:

1. **File Deployment** (for create/update/rename):
   - Use FileTransfer method pattern to copy file from git tree to systemd directory
   - Preserve file permissions
   - Create destination directory if missing

2. **Systemd Daemon Reload**:
   - Execute `systemctl daemon-reload` (root) or `systemctl --user daemon-reload` (user)
   - Required after any file changes to trigger Quadlet generator
   - Use helper container pattern from Systemd method

3. **Service Management**:
   - Service name derived from filename: `webapp.container` → `webapp.service`
   - Enable: `systemctl enable {service}` (if Enable=true)
   - Start: `systemctl start {service}` (if Enable=true on create)
   - Restart: `systemctl restart {service}` (if Restart=true on update)
   - Stop: `systemctl stop {service}` (on delete/rename)

**Error Handling**:
- Return descriptive errors for each failure point
- Include file path, service name, and command that failed
- Log errors before returning
- Do not silently swallow errors

---

## Helper Method Contracts

### quadletPodman(ctx, conn context.Context, path, dest string, prev, curr *string, changeType *string) error

**Purpose**: Internal helper method that executes Quadlet-specific deployment logic.

**Contract**:
- **Input**:
  - `ctx`: Context for operations
  - `conn`: Podman connection context
  - `path`: Source file path in git working tree
  - `dest`: Destination directory (/etc/containers/systemd or ~/.config/containers/systemd)
  - `prev`: Previous filename (for rename/delete), may be nil
  - `curr`: Current filename (for create/update/rename), may be nil
  - `changeType`: Pointer to change type string ("create", "update", "rename", "delete")
- **Output**: Error if any operation fails
- **Behavior**:
  - Delegates to FileTransfer for file operations
  - Calls `systemctlDaemonReload()` after file changes
  - Calls `systemctlManageService()` for service lifecycle operations

### systemctlDaemonReload(conn context.Context, dest string, root bool) error

**Purpose**: Execute systemctl daemon-reload to trigger Quadlet generator.

**Contract**:
- **Input**:
  - `conn`: Podman connection context
  - `dest`: Systemd directory path
  - `root`: Boolean indicating root or user mode
- **Output**: Error if daemon-reload fails
- **Behavior**:
  - Create privileged helper container (similar to Systemd method)
  - Mount systemd socket, cgroup filesystem, and Quadlet directory
  - Execute `systemctl daemon-reload` or `systemctl --user daemon-reload`
  - Wait for completion and remove container
  - Return error if exit code non-zero

### systemctlManageService(conn context.Context, action, service string, root bool) error

**Purpose**: Execute systemctl commands to manage Quadlet-generated services.

**Contract**:
- **Input**:
  - `conn`: Podman connection context
  - `action`: Command action ("enable", "start", "restart", "stop", "disable")
  - `service`: Service name (e.g., "webapp.service")
  - `root`: Boolean indicating root or user mode
- **Output**: Error if systemctl command fails
- **Behavior**:
  - Create privileged helper container
  - Mount systemd socket and cgroup filesystem
  - Execute `systemctl {action} {service}` or `systemctl --user {action} {service}`
  - Capture stdout/stderr
  - Return error if exit code non-zero

---

## Integration Points

### With FetchitConfig

**Registration in getMethodTargetScheds()** (fetchit.go):
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

### With Scheduler

**Job Registration** (RunTargets in fetchit.go):
```go
for method, schedInfo := range f.methodTargetScheds {
    skew := 0
    if schedInfo.skew != nil {
        skew = rand.Intn(*schedInfo.skew)
    }
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    mt := method.GetKind()  // Returns "quadlet"
    logger.Infof("Processing git target: %s Method: %s Name: %s", method.GetTarget().url, mt, method.GetName())
    s.Cron(schedInfo.schedule).Tag(mt).Do(method.Process, ctx, f.conn, skew)
    s.StartImmediately()
}
```

### With Git Operations

**Change Detection** (applyChanges in apply.go):
- Computes diff between two commit hashes
- Filters by TargetPath: Only files under specified directory
- Filters by Glob: Matches pattern if specified
- Filters by Tags: Only `.container`, `.volume`, `.network` extensions

**Git Tag Management**:
- `getCurrent()`: Reads `refs/tags/current-quadlet-{name}`
- `updateCurrent()`: Updates tag to latest commit hash
- Tags stored in git repository, persisted across restarts

---

## Error Handling Contract

**Error Return Policy**:
- `GetName()`, `GetKind()`, `GetTarget()`: Never return errors (panic if misconfigured)
- `Process()`: Log errors, do not return (scheduler expects void)
- `Apply()`: Return first error encountered, fail-fast
- `MethodEngine()`: Return detailed error with context

**Error Message Format**:
```
"Failed to {operation} for Quadlet target '{name}': {underlying_error}"
```

**Examples**:
```
"Failed to deploy Quadlet file webapp.container: permission denied"
"Failed to execute systemctl daemon-reload: exit code 1"
"Failed to clone repository https://github.com/example/repo: authentication failed"
```

**Logging Requirements**:
- Log all operations at INFO level (deployments, daemon-reloads, service actions)
- Log change detection at DEBUG level
- Log errors at ERROR level with full context
- Include: target name, file names, commit hashes, commands executed

---

## Testing Contract

**Unit Tests Required**:
- GetName(), GetKind(), GetTarget() return correct values
- Change type determination logic (create/update/rename/delete)
- Destination directory calculation (root vs user)
- Service name derivation from filename

**Integration Tests Required**:
- Process() with mock git repository
- Apply() with real git diff
- MethodEngine() with mock systemctl commands
- Full workflow: clone → deploy → update → delete

**Mock Interfaces**:
- Mock Podman connection for unit tests
- Mock systemctl commands to avoid requiring systemd in CI
- Mock git repository with test Quadlet files

---

## Performance Contract

**Expected Performance**:
- Single file deployment: < 1 second (file copy + daemon-reload)
- Daemon-reload: < 5 seconds
- Service start: < 10 seconds (depends on container image pull)
- Change detection: < 1 second for typical repository size

**Scalability**:
- Support 100+ Quadlet files per target
- Support 10+ concurrent Quadlet targets
- Mutex ensures no concurrent operations on same target

---

## Backwards Compatibility Contract

**No Breaking Changes**:
- Existing methods (Raw, Systemd, Kube, Ansible, FileTransfer) unaffected
- Existing config.yaml files without `quadlet:` section continue to work
- Method interface unchanged

**Additive Changes Only**:
- New `Quadlet` field in `TargetConfig`
- New `quadletMethod` constant
- New Quadlet implementation file (quadlet.go)

---

## Summary

The Quadlet method implementation must:
1. Implement all six Method interface methods
2. Follow established patterns from Systemd method
3. Integrate with existing git change detection infrastructure
4. Use helper containers for systemctl operations
5. Handle all four change types: create, update, rename, delete
6. Support both root and user deployment modes
7. Log comprehensively for troubleshooting
8. Return descriptive errors with full context

This contract ensures the Quadlet method integrates seamlessly with FetchIt's architecture while providing robust, reliable Quadlet file deployment through systemd.
