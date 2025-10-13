# System Integration Contract: Quadlet Method

**Feature**: FetchIt Modernization - Quadlet Method System Integration
**Date**: 2025-10-13
**Status**: Design Complete

---

## Overview

This document specifies the integration contract between the FetchIt Quadlet method and external system components: systemd, Podman API, and the host filesystem. It defines the exact commands, API calls, file operations, and permission requirements needed for successful Quadlet deployment.

---

## systemd Integration

### Daemon Reload Operations

**Purpose**: Trigger Quadlet generator to process .container, .volume, and .network files and generate corresponding systemd service units.

#### Root Mode Daemon Reload

**Command**:
```bash
systemctl daemon-reload
```

**When Executed**:
- After creating new Quadlet file
- After updating existing Quadlet file
- After deleting Quadlet file
- After renaming Quadlet file

**Expected Behavior**:
- systemd scans `/etc/containers/systemd/` for Quadlet files
- Quadlet generator (podman-systemd-generator) processes files
- Generated `.service` files placed in `/run/systemd/generator/`
- systemd unit database updated

**Exit Codes**:
- `0`: Success, daemon reloaded
- `1`: General error (permissions, systemd not running)

**Execution Context**:
- Must run as root
- Requires systemd running
- Requires `/run/systemd/system/` writable

#### User Mode Daemon Reload

**Command**:
```bash
systemctl --user daemon-reload
```

**When Executed**: Same as root mode

**Expected Behavior**:
- systemd scans `~/.config/containers/systemd/` for Quadlet files
- Generated files placed in `$XDG_RUNTIME_DIR/systemd/generator/`
- User systemd instance database updated

**Exit Codes**: Same as root mode

**Execution Context**:
- Runs as non-root user
- Requires `$XDG_RUNTIME_DIR` set (typically `/run/user/$UID`)
- Requires user systemd instance running
- Requires `~/.config/containers/systemd/` writable

**Environment Variables Required**:
- `XDG_RUNTIME_DIR`: User runtime directory (e.g., `/run/user/1000`)
- `HOME`: User home directory (e.g., `/home/username`)
- `DBUS_SESSION_BUS_ADDRESS`: D-Bus session bus address (set by systemd)

---

### Service Management Operations

#### Enable Service (Auto-start on Boot)

**Root Mode**:
```bash
systemctl enable <service-name>.service
```

**User Mode**:
```bash
systemctl --user enable <service-name>.service
```

**When Executed**:
- After creating new Quadlet file (if `Enable=true`)
- After renaming Quadlet file (if `Enable=true`)

**Expected Behavior**:
- Creates symlinks in appropriate `.wants/` directories
- Service will start automatically on system/user boot
- Does not start service immediately (use `enable --now` to start)

**Exit Codes**:
- `0`: Success, service enabled
- `1`: Service unit not found
- `5`: Service already enabled (not an error for FetchIt)

**Service Name Derivation**:
- Quadlet file: `webapp.container`
- Service name: `webapp.service`
- Formula: `{filename_without_extension}.service`

#### Start Service

**Root Mode**:
```bash
systemctl start <service-name>.service
```

**User Mode**:
```bash
systemctl --user start <service-name>.service
```

**When Executed**:
- After enabling service on initial deployment (if `Enable=true`)
- Can be combined with enable: `systemctl enable --now <service>`

**Expected Behavior**:
- Quadlet-generated service starts
- Podman container created and started
- Service enters `active (running)` state

**Exit Codes**:
- `0`: Success, service started
- `1`: Service failed to start (check `journalctl` for details)

#### Restart Service

**Root Mode**:
```bash
systemctl restart <service-name>.service
```

**User Mode**:
```bash
systemctl --user restart <service-name>.service
```

**When Executed**:
- After updating Quadlet file (if `Restart=true`)

**Expected Behavior**:
- Stops existing container
- Starts new container with updated configuration
- Brief service interruption (downtime)

**Exit Codes**: Same as start

#### Stop Service

**Root Mode**:
```bash
systemctl stop <service-name>.service
```

**User Mode**:
```bash
systemctl --user stop <service-name>.service
```

**When Executed**:
- Before deleting Quadlet file
- Before renaming Quadlet file (stop old service)

**Expected Behavior**:
- Graceful container stop (SIGTERM, then SIGKILL)
- Service enters `inactive (dead)` state
- Container removed by Podman

**Exit Codes**:
- `0`: Success, service stopped
- `5`: Service already stopped (not an error)

#### Disable Service

**Root Mode**:
```bash
systemctl disable <service-name>.service
```

**User Mode**:
```bash
systemctl --user disable <service-name>.service
```

**When Executed**:
- Before permanently removing Quadlet file (optional)

**Expected Behavior**:
- Removes symlinks from `.wants/` directories
- Service will not auto-start on next boot
- Does not stop currently running service

**Exit Codes**:
- `0`: Success, service disabled
- `5`: Service not enabled (not an error)

---

### Status Verification Operations

#### Check Service Status

**Root Mode**:
```bash
systemctl status <service-name>.service
```

**User Mode**:
```bash
systemctl --user status <service-name>.service
```

**When Used**: For verification and troubleshooting

**Output Includes**:
- Service state: `active (running)`, `inactive (dead)`, `failed`
- PID of main process
- Recent log entries
- Enabled/disabled status

**Exit Codes**:
- `0`: Service active (running)
- `1`: Service dead/failed
- `3`: Service inactive
- `4`: Service not found

#### List Service Units

**Root Mode**:
```bash
systemctl list-units --type=service --all | grep <service-name>
```

**User Mode**:
```bash
systemctl --user list-units --type=service --all | grep <service-name>
```

**When Used**: To verify Quadlet-generated services exist

---

### systemctl Execution via Helper Container

**Pattern** (from Systemd method in FetchIt):

FetchIt runs in a container and cannot directly execute `systemctl` on the host. Solution: Create privileged helper container with systemd access.

**Container Specification**:
```go
s := specgen.NewSpecGenerator("quay.io/fetchit/fetchit-systemd:latest", false)
s.Privileged = true
s.PidNS = specgen.Namespace{NSMode: "host"}

// Mounts (root mode)
s.Mounts = []specs.Mount{
    {Source: "/etc/containers/systemd", Destination: "/etc/containers/systemd", Type: "bind", Options: []string{"rw"}},
    {Source: "/run", Destination: "/run", Type: "tmpfs", Options: []string{"rw"}},
    {Source: "/sys/fs/cgroup", Destination: "/sys/fs/cgroup", Type: "bind", Options: []string{"ro"}},
    {Source: "/run/systemd", Destination: "/run/systemd", Type: "bind", Options: []string{"rw"}},
}

// Environment variables
s.Env = map[string]string{
    "ROOT": "true",  // or "false" for user mode
    "SERVICE": "webapp.service",
    "ACTION": "enable",  // or "start", "restart", "stop", "daemon-reload"
    "HOME": os.Getenv("HOME"),  // User mode only
    "XDG_RUNTIME_DIR": "/run/user/1000",  // User mode only
}
```

**Mounts Required (User Mode)**:
```go
runMountsd := os.Getenv("XDG_RUNTIME_DIR") + "/systemd"  // e.g., /run/user/1000/systemd
runMounttmp := os.Getenv("XDG_RUNTIME_DIR")  // e.g., /run/user/1000

s.Mounts = []specs.Mount{
    {Source: "$HOME/.config/containers/systemd", Destination: "$HOME/.config/containers/systemd", Type: "bind", Options: []string{"rw"}},
    {Source: runMounttmp, Destination: runMounttmp, Type: "tmpfs", Options: []string{"rw"}},
    {Source: "/sys/fs/cgroup", Destination: "/sys/fs/cgroup", Type: "bind", Options: []string{"ro"}},
    {Source: runMountsd, Destination: runMountsd, Type: "bind", Options: []string{"rw"}},
}
```

**Container Lifecycle**:
1. Create container with mounts and environment
2. Start container
3. Wait for completion
4. Capture exit code and logs
5. Remove container

---

## Podman API Integration

The Quadlet method uses Podman Go bindings for container operations related to systemctl execution.

### Podman Bindings Used

**Package**: `github.com/containers/podman/v4/pkg/bindings`

#### Connection Establishment

**Function**: `bindings.NewConnection(ctx context.Context, uri string) (context.Context, error)`

**Usage**:
```go
conn, err := bindings.NewConnection(ctx, "unix://run/podman/podman.sock")
```

**Connection URI**:
- Root: `unix://run/podman/podman.sock`
- User: `unix://${XDG_RUNTIME_DIR}/podman/podman.sock` (e.g., `unix:///run/user/1000/podman/podman.sock`)

**Error Handling**:
- Connection failed: Podman service not running
- Permission denied: Socket not accessible
- Not found: Socket path incorrect

#### Container Creation

**Function**: `containers.CreateWithSpec(conn context.Context, s *specgen.SpecGenerator, options *containers.CreateOptions) (*entities.ContainerCreateResponse, error)`

**Usage**:
```go
createResponse, err := containers.CreateWithSpec(conn, specgen, nil)
if err != nil {
    return fmt.Errorf("Failed to create helper container: %w", err)
}
containerID := createResponse.ID
```

**Response**:
- `ID`: Container ID (64-character hex string)
- `Warnings`: Array of warning messages

#### Container Start

**Function**: `containers.Start(conn context.Context, nameOrID string, options *containers.StartOptions) error`

**Usage**:
```go
err := containers.Start(conn, containerID, nil)
if err != nil {
    return fmt.Errorf("Failed to start helper container: %w", err)
}
```

#### Container Wait

**Function**: `containers.Wait(conn context.Context, nameOrID string, options *containers.WaitOptions) (int32, error)`

**Usage**:
```go
exitCode, err := containers.Wait(conn, containerID, &containers.WaitOptions{
    Condition: []define.ContainerStatus{define.ContainerStateStopped},
})
if err != nil {
    return fmt.Errorf("Error waiting for container: %w", err)
}
if exitCode != 0 {
    return fmt.Errorf("systemctl command failed with exit code %d", exitCode)
}
```

**Wait Conditions**:
- `define.ContainerStateStopped`: Wait until container stops
- `define.ContainerStateExited`: Wait until container exits

#### Container Logs

**Function**: `containers.Logs(conn context.Context, nameOrID string, options *containers.LogOptions) error`

**Usage**:
```go
logOptions := &containers.LogOptions{
    Stdout: true,
    Stderr: true,
    Follow: false,
}
logs, err := containers.Logs(conn, containerID, logOptions)
```

**Use Case**: Capture stdout/stderr from systemctl commands for error reporting

#### Container Remove

**Function**: `containers.Remove(conn context.Context, nameOrID string, options *containers.RemoveOptions) (*entities.ContainerRemoveReport, error)`

**Usage**:
```go
_, err := containers.Remove(conn, containerID, &containers.RemoveOptions{
    Force: true,
})
```

**Options**:
- `Force`: Remove even if running (not needed, container already stopped)
- `Volumes`: Remove anonymous volumes (not applicable)

---

### Image Operations

#### Image Detection/Pull

**Function**: `images.Exists(conn context.Context, nameOrID string, options *images.ExistsOptions) (bool, error)`

**Usage**:
```go
exists, err := images.Exists(conn, "quay.io/fetchit/fetchit-systemd:latest", nil)
if !exists {
    // Pull image
    _, err := images.Pull(conn, "quay.io/fetchit/fetchit-systemd:latest", &images.PullOptions{})
}
```

**Images Required**:
- `quay.io/fetchit/fetchit-systemd:latest`: Helper container with systemctl
- Contains: systemd client tools, minimal Alpine/UBI base

---

## Filesystem Integration

### Quadlet File Deployment Locations

#### Root Mode

**Directory**: `/etc/containers/systemd/`

**Permissions**:
- Owner: `root:root`
- Mode: `0755` (drwxr-xr-x)
- Files: `0644` (-rw-r--r--)

**Required Operations**:
- Check directory exists: `stat /etc/containers/systemd/`
- Create if missing: `mkdir -p /etc/containers/systemd/`
- Write Quadlet file: `cp /path/in/git/webapp.container /etc/containers/systemd/webapp.container`
- Remove Quadlet file: `rm /etc/containers/systemd/webapp.container`

**File Types**:
- `*.container`: Container definitions
- `*.volume`: Volume definitions
- `*.network`: Network definitions

#### User Mode

**Directory**: `$HOME/.config/containers/systemd/`

**Permissions**:
- Owner: `{user}:{group}` (current user)
- Mode: `0755` (drwxr-xr-x)
- Files: `0644` (-rw-r--r--)

**Required Operations**: Same as root mode, but user-writable

**Environment Dependencies**:
- `$HOME` must be set
- `$HOME/.config/` must exist and be writable
- `$XDG_RUNTIME_DIR` must be set for user systemd

---

### Generated Service File Locations

**Not Managed by FetchIt**: These are created by Quadlet generator during `daemon-reload`

#### Root Mode

**Directory**: `/run/systemd/generator/`

**Example**:
- Quadlet file: `/etc/containers/systemd/webapp.container`
- Generated service: `/run/systemd/generator/webapp.service`

#### User Mode

**Directory**: `$XDG_RUNTIME_DIR/systemd/generator/`

**Example**:
- Quadlet file: `~/.config/containers/systemd/webapp.container`
- Generated service: `/run/user/1000/systemd/generator/webapp.service`

---

### Git Repository Integration

**Clone Location**: `/opt/volume/{repository-name}/`

**Examples**:
- Repository: `https://github.com/company/infra-configs`
- Clone path: `/opt/volume/infra-configs/`
- Quadlet files: `/opt/volume/infra-configs/quadlet/webapp.container`

**Operations**:
- Clone: `git.PlainClone(path, false, cloneOptions)`
- Fetch: `repo.Fetch(&git.FetchOptions{})`
- Worktree: `repo.Worktree()`
- Checkout: `worktree.Checkout(&git.CheckoutOptions{Hash: commitHash})`
- Read file: `ioutil.ReadFile(filepath.Join(clonePath, targetPath, filename))`

**File Copy Pattern** (using FileTransfer method):
```go
ft := &FileTransfer{
    CommonMethod: CommonMethod{
        Name: q.Name,
    },
}
err := ft.fileTransferPodman(ctx, conn, sourcePath, destPath, prevFilename)
```

---

### File Operations Contract

#### Create Quadlet File

**Operation**: Copy file from git working tree to systemd directory

**Steps**:
1. Read file from git: `/opt/volume/infra-configs/quadlet/webapp.container`
2. Ensure destination directory exists: `/etc/containers/systemd/` or `~/.config/containers/systemd/`
3. Write file to destination: `/etc/containers/systemd/webapp.container`
4. Set permissions: `chmod 0644`
5. Set ownership: `chown root:root` (root mode) or current user (user mode)

**Error Cases**:
- Source file not found: Log error, skip file
- Destination not writable: Return error
- Disk full: Return error

#### Update Quadlet File

**Operation**: Replace existing file with updated version

**Steps**: Same as create (overwrites existing file)

**Atomic Update**: Not required (systemd reads on daemon-reload, not continuously)

#### Delete Quadlet File

**Operation**: Remove file from systemd directory

**Steps**:
1. Verify file exists: `stat /etc/containers/systemd/webapp.container`
2. Remove file: `rm /etc/containers/systemd/webapp.container`
3. Return success even if file doesn't exist (idempotent)

**Error Cases**:
- Permission denied: Return error
- File in use: Not possible (systemd doesn't lock files)

#### Rename Quadlet File

**Operation**: Remove old file, create new file

**Steps**:
1. Stop old service: `systemctl stop old-webapp.service`
2. Remove old file: `rm /etc/containers/systemd/old-webapp.container`
3. Create new file: (same as create operation)
4. daemon-reload
5. Enable/start new service: `systemctl enable --now new-webapp.service`

---

## Permission Requirements

### Root Mode Permissions

**FetchIt Process**:
- Must run as root or with `CAP_SYS_ADMIN` capability
- Must have write access to `/etc/containers/systemd/`
- Must be able to create privileged containers

**Helper Container**:
- Must run with `privileged: true`
- Must have host PID namespace access
- Must mount `/run/systemd/` (read-write)
- Must mount `/sys/fs/cgroup/` (read-only)

**systemctl Commands**:
- Must run as root within helper container
- Communicates with systemd via D-Bus

### User Mode Permissions

**FetchIt Process**:
- Can run as non-root user
- Must have write access to `~/.config/containers/systemd/`
- Must have `XDG_RUNTIME_DIR` set

**Helper Container**:
- Must run with user's UID/GID
- Must have `privileged: true` (for systemd access)
- Must mount `$XDG_RUNTIME_DIR/systemd/` (read-write)
- Must mount `$HOME/.config/containers/systemd/` (read-write)

**systemctl Commands**:
- Must run as user within helper container
- Uses `systemctl --user` commands
- Communicates with user systemd instance via D-Bus

**User Lingering**:
- Recommended: `loginctl enable-linger $USER`
- Without lingering: Services stop when user logs out
- With lingering: Services persist across logouts

---

## Required Capabilities

### Linux Capabilities (for helper container)

- `CAP_SYS_ADMIN`: Required for systemd operations
- `CAP_SYS_PTRACE`: May be required for systemd inspection
- `CAP_NET_ADMIN`: Not required for Quadlet method
- `CAP_CHOWN`: For setting file ownership (root mode)

### SELinux Context (if SELinux enabled)

**Quadlet Files**:
- Root: Context `system_u:object_r:container_config_t:s0`
- User: Context `unconfined_u:object_r:user_home_t:s0`

**Helper Container**:
- Label: `system_u:system_r:container_t:s0` (or unconfined)
- Volume mounts: Use `:Z` or `:z` options if needed

**Not Critical**: Quadlet method should work with SELinux permissive or disabled, document any SELinux requirements

---

## Error Conditions and Recovery

### systemd Not Available

**Detection**:
```bash
systemctl --version
# Exit code 127: Command not found
```

**Error Message**:
```
"Quadlet method requires systemd. systemctl command not found on host. Consider using Raw or Kube methods instead."
```

**Recovery**: User must install systemd or switch to different method

### Podman Version Too Old

**Detection**:
```go
version, _ := system.Version(conn, nil)
if version.Server.Version < "4.4.0" {
    return error
}
```

**Error Message**:
```
"Quadlet requires Podman 4.4 or later. Found: {version}. Please upgrade Podman."
```

**Recovery**: User must upgrade Podman

### cgroup v1 Detected

**Detection**:
```bash
podman info --format '{{.Host.CgroupsVersion}}'
# Output: v1
```

**Error Message**:
```
"Quadlet requires cgroup v2. Found: v1. See https://docs.podman.io/en/latest/markdown/podman-system-migrate.1.html for migration instructions."
```

**Recovery**: User must migrate to cgroup v2

### Permission Denied

**Scenario**: FetchIt cannot write to systemd directory

**Error Message**:
```
"Failed to deploy Quadlet file {filename}: permission denied. Ensure FetchIt has write access to {directory}."
```

**Recovery**: User must adjust permissions or run FetchIt with appropriate privileges

### Service Failed to Start

**Scenario**: systemctl start returns non-zero exit code

**Error Message**:
```
"Failed to start service {service}: {systemctl stderr}. Check: journalctl -u {service}"
```

**Recovery**:
1. Check Quadlet file syntax
2. Check container image exists: `podman pull {image}`
3. Check logs: `journalctl -u {service}`
4. Check systemd status: `systemctl status {service}`

---

## Integration Testing Contract

### Test Environment Requirements

- Podman 4.9+ installed
- systemd running (PID 1 or user instance)
- cgroup v2 enabled
- Test git repository with sample Quadlet files
- Write access to systemd directories

### Test Scenarios

1. **Deploy .container file**: Verify file copied, daemon-reloaded, service started
2. **Update .container file**: Verify file updated, daemon-reloaded, service restarted
3. **Delete .container file**: Verify service stopped, file removed
4. **Deploy .volume file**: Verify volume created
5. **Deploy .network file**: Verify network created
6. **Multi-file deployment**: Verify dependencies handled correctly
7. **User mode deployment**: Verify works without root
8. **Error handling**: Verify graceful failure on invalid files

---

## Summary

This contract defines:
- Exact systemctl commands and their execution context
- Podman API bindings used for container operations
- Filesystem paths and permissions for Quadlet files
- Helper container pattern for systemctl execution
- Error conditions and recovery strategies
- Integration testing requirements

Adherence to this contract ensures reliable, secure, and maintainable Quadlet deployment through FetchIt.
