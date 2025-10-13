# Modernize FetchIt: Dependency Updates and Quadlet Method Support

**Feature Overview:**

FetchIt is a GitOps tool for managing Podman-managed containers that currently supports multiple deployment methods (Raw, Systemd, Kube, Ansible, FileTransfer). This RFE proposes two critical enhancements: (1) updating all project dependencies to current stable versions to ensure security, compatibility, and access to modern features, and (2) adding support for Podman Quadlets as a new deployment method. Quadlets represent the modern, declarative approach to managing containers with systemd in Podman 4.4+, replacing the deprecated `podman generate systemd` command. Together, these changes will modernize FetchIt's foundation and extend its capabilities to support contemporary container orchestration patterns.

**Goals:**

* **Dependency Modernization**: Update FetchIt's Go version (currently 1.17 from 2021) and all dependencies to current stable releases, ensuring the project benefits from security patches, performance improvements, bug fixes, and modern Go language features. This includes updating core dependencies like podman/v4, containers/common, go-git, and associated libraries.

* **Quadlet Method Implementation**: Introduce a new "Quadlet" deployment method that enables users to manage containers using Podman's native Quadlet unit files (.container, .volume, .network, .kube, .pod). This provides a declarative, systemd-integrated approach that aligns with Podman's strategic direction and eliminates the complexity of manually managing systemd unit files.

* **Enhanced User Experience**: Users will benefit from improved stability, security, and access to modern Podman features. The Quadlet method will simplify container lifecycle management for users who want systemd integration without the complexity of traditional unit files, while the dependency updates ensure compatibility with contemporary container registries, authentication mechanisms, and orchestration patterns.

* **Future-Proofing**: By modernizing dependencies and adopting Quadlets, FetchIt positions itself as a forward-compatible GitOps solution that aligns with the Podman ecosystem's evolution, ensuring long-term maintainability and community support.

**Out of Scope:**

* Migration tools or automated conversion from existing methods to Quadlet (users can continue using existing methods)
* Breaking changes to existing method configurations (Raw, Systemd, Kube, Ansible, FileTransfer remain backwards compatible)
* Support for other container runtimes beyond Podman
* UI/dashboard for managing FetchIt configurations
* Multi-cluster or remote Podman socket management
* Custom Quadlet syntax extensions beyond what Podman supports natively

**Requirements:**

**MVP Requirements:**

1. **Dependency Update**:
   - Update Go version to 1.21 or later (MVP)
   - Update `github.com/containers/podman/v4` to v4.9+ or v5.x if stable (MVP)
   - Update `github.com/containers/common` to compatible latest version (MVP)
   - Update `github.com/go-git/go-git/v5` to address known security vulnerabilities (MVP)
   - Update all transitive dependencies with known CVEs (MVP)
   - Ensure `go.mod` and `go.sum` are properly updated (MVP)
   - Verify build compatibility across supported platforms (AMD64, ARM64) (MVP)

2. **Quadlet Method Core Implementation**:
   - Implement new `Quadlet` type in `pkg/engine/quadlet.go` following Method interface pattern (MVP)
   - Support `.container` file deployment and lifecycle management (MVP)
   - Support `.volume` file deployment for persistent storage (MVP)
   - Support `.network` file deployment for custom networking (MVP)
   - Integrate Quadlet processing into git change detection and apply logic (MVP)
   - Enable scheduling for Quadlet targets via cron expressions (MVP)
   - Add Quadlet configuration to TargetConfig structure in types.go (MVP)

3. **Quadlet Method Configuration**:
   - Define YAML configuration schema for Quadlet method in config.yaml (MVP)
   - Support targetPath, schedule, glob patterns for Quadlet files (MVP)
   - Implement proper file filtering for .container, .volume, .network extensions (MVP)

4. **Testing and Validation**:
   - Create example Quadlet configurations in examples/ directory (MVP)
   - Verify Quadlet units are properly deployed to systemd paths (MVP)
   - Test integration with podman-systemd-unit functionality (MVP)
   - Ensure clean updates when Quadlet files change in git (MVP)

**Non-MVP Requirements:**

5. **Extended Quadlet Support**:
   - Support `.kube` Quadlet files for Kubernetes YAML deployment
   - Support `.pod` Quadlet files for Podman pod management
   - Support `.build` Quadlet files for image building
   - Support `.image` Quadlet files for image pulling
   - Advanced Quadlet dependency management and ordering

6. **Enhanced Dependency Management**:
   - Automated dependency update monitoring (e.g., Dependabot)
   - Comprehensive security scanning integration
   - Vendor directory management strategies

7. **Documentation and Migration**:
   - Comprehensive Quadlet usage guide in docs/
   - Migration guide from Systemd method to Quadlet method
   - Performance comparison documentation

**Done - Acceptance Criteria:**

The feature is complete when:

1. **Dependency Updates Verified**:
   - All dependencies in go.mod are updated to compatible latest stable versions
   - Go version is 1.21+ and verified in CI/CD pipelines
   - All existing methods (Raw, Systemd, Kube, Ansible, FileTransfer) continue to function without regression
   - No high or critical severity CVEs remain in dependency tree
   - Build succeeds on AMD64 and ARM64 architectures
   - Existing example configurations in examples/ directory work without modification

2. **Quadlet Method Operational**:
   - Users can define Quadlet targets in config.yaml with name, targetPath, schedule, and glob fields
   - FetchIt successfully clones repos, detects .container/.volume/.network files, and deploys them to appropriate systemd directories
   - Quadlet files are placed in correct locations (root: /etc/containers/systemd/ or user: ~/.config/containers/systemd/)
   - Changes to Quadlet files in git trigger proper systemd daemon reloads and service restarts
   - Container services defined via Quadlet start successfully and persist across reboots
   - Quadlet-managed containers can be inspected via `podman ps` and `systemctl status`

3. **Documentation Complete**:
   - README.md updated to mention Go version and Quadlet support
   - docs/methods.rst includes Quadlet method with configuration examples
   - examples/quadlet/ directory contains working .container, .volume, .network sample files
   - CHANGELOG or release notes document dependency updates and new Quadlet method

4. **Quality Assurance**:
   - Automated tests pass for all methods including new Quadlet method
   - Manual testing confirms Quadlet containers run as systemd services
   - No memory leaks or resource exhaustion in long-running scenarios
   - Error handling gracefully reports issues with malformed Quadlet files

**Use Cases - i.e. User Experience & Workflow:**

**Use Case 1: Deploying Containers with Quadlet Method**

*Main Success Scenario:*
1. User creates a git repository containing Quadlet unit files (.container, .volume, .network)
2. User updates FetchIt config.yaml to include Quadlet target:
   ```yaml
   targetConfigs:
   - url: https://github.com/example/my-containers
     branch: main
     quadlet:
     - name: web-services
       targetPath: quadlets/
       root: true
       schedule: "*/5 * * * *"
   ```
3. FetchIt clones the repository and detects Quadlet files in quadlets/ directory
4. FetchIt deploys .container, .volume, .network files to /etc/containers/systemd/
5. FetchIt triggers `systemctl daemon-reload` to register new units
6. Containers start automatically via systemd based on Quadlet definitions
7. User verifies with `systemctl status <service>` and `podman ps`
8. When user updates .container file in git, FetchIt detects change on next schedule
9. FetchIt updates the systemd unit and restarts the service
10. Container runs with new configuration

*Alternative Flow - Non-Root User:*
- If `root: false`, Quadlet files deploy to ~/.config/containers/systemd/
- Services run in user systemd context via `systemctl --user`

**Use Case 2: Updating Dependencies for Security Patches**

*Main Success Scenario:*
1. Maintainer reviews current dependencies and identifies outdated versions with CVEs
2. Maintainer updates go.mod with `go get -u` commands for targeted dependencies
3. Maintainer runs `go mod tidy` and `go mod vendor` to update vendored code
4. CI/CD pipeline runs automated tests against updated dependencies
5. Tests pass, verifying no breaking changes in existing methods
6. Maintainer builds FetchIt container image with updated dependencies
7. Users pull updated FetchIt image and benefit from security fixes
8. No configuration changes required by end users

**Use Case 3: Migrating from Systemd Method to Quadlet Method**

*Main Success Scenario:*
1. User currently uses Systemd method with traditional .service files
2. User reads migration documentation and learns about Quadlet advantages
3. User converts .service files to .container Quadlet format (simplified syntax)
4. User updates config.yaml to change `systemd:` section to `quadlet:` section
5. User commits new Quadlet files to git repository
6. FetchIt detects new configuration and deploys Quadlet files
7. User verifies containers run correctly with new Quadlet method
8. User removes old systemd method configuration after validation period

**Use Case 4: Complex Multi-Container Application with Dependencies**

*Main Success Scenario:*
1. User has application requiring database container, app container, and shared volume
2. User creates myapp.volume Quadlet for persistent data storage
3. User creates myapp-db.container Quadlet with volume reference
4. User creates myapp-web.container Quadlet with database network dependency
5. User commits all files to git repository under applications/ directory
6. FetchIt deploys all Quadlet files to systemd
7. Systemd starts services in correct order based on dependencies
8. Application runs successfully with database and volume connectivity

**Documentation Considerations:**

1. **New Documentation Required**:
   - Comprehensive Quadlet method guide in docs/quadlet.rst or docs/methods.rst
   - Quadlet file format reference with examples for .container, .volume, .network
   - Troubleshooting guide for common Quadlet deployment issues
   - Comparison table: Raw vs. Systemd vs. Quadlet methods
   - Security best practices for Quadlet configurations

2. **Updated Documentation**:
   - Update docs/running.rst to reflect new Go version requirements
   - Update README.md with Quadlet example in launch options
   - Update examples/ directory with new examples/quadlet/ folder
   - Update docs/methods.rst to add Quadlet section alongside existing methods

3. **Dependency Update Documentation**:
   - Document breaking changes (if any) in dependency updates
   - Provide migration guide for users on older FetchIt versions
   - Document new features available from updated Podman libraries
   - Update Dockerfile and Dockerfile-ubuntu with new base image versions if needed

4. **Existing Documentation Links**:
   - Current methods documentation: [docs/methods.rst](https://github.com/containers/fetchit/blob/main/docs/methods.rst)
   - Existing examples: [examples/](https://github.com/containers/fetchit/tree/main/examples)
   - Quick start guide: [docs/quick_start.rst](https://github.com/containers/fetchit/blob/main/docs/quick_start.rst)

**Questions to answer:**

1. **Dependency Management**:
   - What is the target Go version? (1.21, 1.22, or 1.23?)
   - Should we update to Podman v5 bindings if available, or stay on v4.x?
   - Are there any known breaking changes in newer versions of go-git or other core dependencies?
   - What is the policy for transitive dependency updates (update all vs. conservative approach)?
   - Should we implement dependency pinning strategies for critical libraries?

2. **Quadlet Implementation**:
   - Should Quadlet method support both root and non-root deployments similar to Systemd method?
   - How should FetchIt handle Quadlet files that reference non-existent volumes or networks? (Create automatically vs. error)
   - Should FetchIt validate Quadlet syntax before deployment, or rely on Podman's validation?
   - How should we handle systemd reload timing to avoid race conditions with multiple Quadlet file changes?
   - Should enable/restart flags be supported for Quadlet method similar to Systemd method?

3. **Backwards Compatibility**:
   - Will dependency updates require any changes to existing method implementations?
   - Should we provide a deprecation notice for older approaches that Quadlet supersedes?
   - What is the migration path for users on older FetchIt versions with incompatible configs?

4. **Testing Strategy**:
   - What automated test coverage is required for Quadlet method (unit, integration, e2e)?
   - How do we test systemd integration in CI/CD environments (containers vs. VMs)?
   - Should we implement smoke tests for all methods after dependency updates?
   - What platforms must be validated (RHEL, Fedora, Ubuntu, CentOS Stream)?

5. **Performance and Scalability**:
   - Are there performance implications of newer dependency versions we should benchmark?
   - How many Quadlet files per target should be supported before performance degrades?
   - Should there be rate limiting or throttling for systemd daemon reloads?

6. **Security and Permissions**:
   - What are the minimum required permissions for deploying Quadlet files (root vs. non-root)?
   - Should FetchIt validate Quadlet files for security anti-patterns before deployment?
   - How should sensitive data in Quadlet files (env vars, secrets) be handled?

7. **Container Image Distribution**:
   - Should separate container images be built for different architectures after updates?
   - What base image versions should be used in Dockerfile updates?
   - Should we maintain compatibility images for users who cannot upgrade immediately?

**Background & Strategic Fit:**

**Project Context:**

FetchIt is a GitOps solution for managing Podman containers on edge and traditional infrastructure, enabling declarative, version-controlled container deployments. It is particularly valuable for environments where Kubernetes is too heavyweight but automated, auditable container management is essential (e.g., IoT edge devices, small-scale servers, development environments).

**Strategic Importance:**

1. **Alignment with Podman Ecosystem Evolution**: Podman 4.4 introduced Quadlets as the recommended way to manage containers with systemd, deprecating `podman generate systemd`. FetchIt must adopt Quadlets to remain aligned with Podman's strategic direction and ensure users benefit from modern tooling.

2. **Security and Compliance**: Outdated dependencies pose security risks and may fail compliance scans in enterprise environments. Updating to Go 1.21+ and modern library versions addresses CVEs, ensures TLS 1.3 support, and provides memory safety improvements.

3. **Developer Experience**: Modern Go versions (1.21+) offer improved error handling, generics, and performance optimizations that improve code maintainability and developer productivity.

4. **Long-Term Maintainability**: As upstream projects (Podman, go-git, containers/common) evolve, staying current with dependencies prevents technical debt accumulation and reduces the cost of future updates.

**Why Quadlets Matter:**

- **Simplified Container Management**: Quadlets abstract away systemd complexity, letting users focus on container configuration rather than unit file mechanics.
- **Declarative and Portable**: Quadlet files are more readable and portable than traditional systemd units, improving GitOps workflows.
- **Native Podman Integration**: Quadlets are first-class citizens in Podman, with better support, documentation, and tooling compared to manually generated units.
- **Automatic Lifecycle Management**: Podman handles systemd integration automatically, reducing maintenance burden for users.

**Community and Ecosystem Benefits:**

- Demonstrates FetchIt's commitment to modern container practices
- Attracts users seeking Quadlet-based GitOps workflows
- Aligns with Red Hat and Fedora ecosystem directions
- Positions FetchIt as a reference implementation for Podman GitOps

**Customer Considerations:**

1. **Enterprise Customers**:
   - Require up-to-date dependencies for security compliance and vulnerability scanning
   - Value systemd integration for enterprise Linux environments (RHEL, CentOS Stream)
   - Need stable, supported tooling with clear upgrade paths
   - May have internal policies requiring Go 1.21+ for TLS and crypto compliance

2. **Edge Computing and IoT**:
   - Benefit from Quadlet's lightweight approach compared to Kubernetes
   - Require reliable, automatic container restarts on edge devices with intermittent connectivity
   - Value GitOps workflows for managing distributed edge infrastructure
   - Need minimal resource overhead, which updated dependencies may improve

3. **Development Teams**:
   - Want simplified container-to-systemd workflows without manual unit file generation
   - Appreciate declarative configuration that version controls cleanly in git
   - Benefit from improved error messages and debugging in newer library versions
   - Need compatibility with modern container registries and authentication methods (OAuth, OCI distribution spec v2)

4. **Migration Considerations**:
   - Existing users with Systemd method should not be forced to migrate immediately
   - Quadlet should be opt-in, allowing gradual adoption
   - Documentation must clearly explain when to use each method (Raw, Systemd, Quadlet, Kube)
   - Backward compatibility is critical to avoid disrupting production deployments

5. **Platform-Specific Concerns**:
   - **RHEL/CentOS Users**: May be conservative with updates; need tested, stable releases
   - **Fedora Users**: Expect cutting-edge features and rapid adoption of new Podman capabilities
   - **Ubuntu Users**: May require additional documentation for systemd user services
   - **ARM64 Users**: Need verified compatibility with updated dependencies on ARM architectures

6. **Operational Requirements**:
   - Zero-downtime updates when switching from one method to another
   - Clear logging and error reporting for Quadlet deployment failures
   - Ability to rollback to previous container states if new Quadlet configs fail
   - Monitoring and observability integration (e.g., systemd journal integration)

7. **Training and Adoption**:
   - Teams familiar with Systemd method may need training on Quadlet syntax and concepts
   - Documentation should include side-by-side comparisons and migration examples
   - Consider providing validation tools to check Quadlet files before committing to git
   - Community support channels should be prepared to answer Quadlet-specific questions

**Dependency Update Risk Assessment**:

- **Low Risk**: Well-tested libraries with strong semver practices (go-git, cobra, viper)
- **Medium Risk**: Podman libraries (may have API changes between versions)
- **Mitigation**: Comprehensive automated testing, staged rollout, rollback documentation
