# Data Model: Documentation Artifact Structure

**Feature**: Update README and Documentation for vTeam
**Phase**: Phase 1 - Design
**Date**: 2025-10-10

---

## Overview

This data model defines the structure of documentation artifacts, their relationships to source code, and the mechanisms for tracking documentation gaps and update requirements.

---

## Entities

### 1. DocumentationArtifact

Represents a single documentation file or section within the vTeam repository.

**Fields**:
- `file_path` (string, required): Absolute path to documentation file (e.g., `/vTeam/README.md`)
- `artifact_type` (enum, required): Type of documentation
  - Values: `README`, `COMPONENT_README`, `USER_GUIDE`, `DEVELOPER_GUIDE`, `API_REFERENCE`, `DEPLOYMENT_GUIDE`, `INTEGRATION_GUIDE`, `CONFIGURATION_REFERENCE`
- `status` (enum, required): Current state of documentation
  - Values: `CURRENT`, `OUTDATED`, `MISSING`, `PARTIALLY_DOCUMENTED`
- `component` (enum, nullable): Associated vTeam component
  - Values: `FRONTEND`, `BACKEND`, `OPERATOR`, `RUNNER`, `RUNNER_SHELL`, `PLATFORM`, `null` (for cross-component docs)
- `priority` (enum, required): Update priority based on impact
  - Values: `P0_CRITICAL`, `P1_HIGH`, `P2_MEDIUM`, `P3_LOW`
- `created_date` (datetime, nullable): When artifact was originally created
- `last_modified_date` (datetime, nullable): When artifact was last updated

**Relationships**:
- `references` → Many SourceCodeFile (what code does this documentation describe?)
- `has_gaps` → Many DocumentationGap (what's wrong with this documentation?)
- `requires_updates` → Many UpdateRequirement (what specific changes are needed?)

**Validation Rules**:
- `file_path` must exist in vTeam repository or be marked as `MISSING`
- If `component` is not null, must correspond to actual directory in `components/`
- `status=CURRENT` requires no open `DocumentationGap` records with `severity=HIGH`

**State Transitions**:
```
MISSING → OUTDATED → CURRENT
          ↑           ↓
          ←──────────
         (code changes without doc updates)
```

**Examples**:
```yaml
- file_path: /vTeam/README.md
  artifact_type: README
  status: OUTDATED
  component: PLATFORM
  priority: P0_CRITICAL

- file_path: /vTeam/components/backend/README.md
  artifact_type: COMPONENT_README
  status: MISSING
  component: BACKEND
  priority: P0_CRITICAL

- file_path: /vTeam/docs/API_REFERENCE.md
  artifact_type: API_REFERENCE
  status: MISSING
  component: BACKEND
  priority: P0_CRITICAL
```

---

### 2. SourceCodeFile

Represents a source code file that contains features or implementation details that should be documented.

**Fields**:
- `file_path` (string, required): Absolute path to source code file
- `component` (enum, required): Component this file belongs to
  - Values: `FRONTEND`, `BACKEND`, `OPERATOR`, `RUNNER`, `RUNNER_SHELL`
- `language` (enum, required): Programming language
  - Values: `TYPESCRIPT`, `GO`, `PYTHON`, `YAML`, `MARKDOWN`
- `features` (array of string, required): List of features implemented in this file
- `api_endpoints` (array of object, nullable): HTTP endpoints defined in this file
  - Object shape: `{method: string, path: string, handler: string, line_number: int}`
- `environment_variables` (array of string, nullable): Environment variables used in this file
- `primary_purpose` (string, required): Main purpose of this file (1-2 sentences)

**Relationships**:
- `documented_by` → Many DocumentationArtifact (what documentation describes this code?)
- `triggers_gaps` → Many DocumentationGap (what documentation gaps does this code expose?)

**Validation Rules**:
- `file_path` must exist in vTeam repository
- `component` must match directory structure (`components/{component}/`)
- At least one feature must be listed in `features` array

**Examples**:
```yaml
- file_path: /vTeam/components/backend/main.go
  component: BACKEND
  language: GO
  features:
    - "HTTP API server initialization"
    - "Dual-mode operation (API vs Content Service)"
    - "64 HTTP endpoint definitions"
    - "CORS configuration"
    - "Kubernetes client initialization"
  api_endpoints:
    - method: GET
      path: /api/projects
      handler: listProjects
      line_number: 209
    - method: POST
      path: /api/projects
      handler: createProject
      line_number: 210
  environment_variables:
    - NAMESPACE
    - PORT
    - CONTENT_SERVICE_MODE
  primary_purpose: "Main entry point for backend API server; defines all HTTP routes and initializes Gin router"

- file_path: /vTeam/components/backend/github_app.go
  component: BACKEND
  language: GO
  features:
    - "GitHub App OAuth flow with HMAC-signed state"
    - "User-to-installation mapping via ConfigMap storage"
    - "Fork management (list/create)"
    - "Repository tree and blob proxy"
  api_endpoints:
    - method: POST
      path: /api/auth/github/install
      handler: linkGitHubInstallationGlobal
      line_number: 203
  environment_variables:
    - GITHUB_CLIENT_ID
    - GITHUB_CLIENT_SECRET
    - GITHUB_STATE_SECRET
  primary_purpose: "Implements GitHub App integration including OAuth, installation tracking, and repository operations"
```

---

### 3. DocumentationGap

Represents a specific gap or inaccuracy in documentation identified during the audit.

**Fields**:
- `gap_id` (string, required): Unique identifier (e.g., `GAP-README-001`)
- `artifact_path` (string, required): Path to documentation artifact with the gap
- `gap_type` (enum, required): Type of documentation gap
  - Values: `MISSING_FEATURE`, `OUTDATED_INFO`, `INCORRECT_EXAMPLE`, `WRONG_COMMAND`, `MISSING_COMPONENT`, `WRONG_NAME`, `MISSING_CONFIG`, `MISSING_ENDPOINT`
- `severity` (enum, required): Impact of this gap
  - Values: `HIGH` (blocks users/developers), `MEDIUM` (causes confusion), `LOW` (minor improvement)
- `description` (string, required): Human-readable description of the gap
- `source_reference` (string, required): Path to code that proves this gap exists (file:line)
- `current_state` (string, nullable): What documentation currently says (if outdated/incorrect)
- `desired_state` (string, required): What documentation should say

**Relationships**:
- `belongs_to` → One DocumentationArtifact
- `references` → One or more SourceCodeFile

**State Transitions**:
```
IDENTIFIED → PRIORITIZED → FIXED → VERIFIED
```

**Examples**:
```yaml
- gap_id: GAP-README-001
  artifact_path: /vTeam/README.md
  gap_type: WRONG_COMMAND
  severity: HIGH
  description: "Troubleshooting command uses wrong deployment name"
  source_reference: /vTeam/components/manifests/operator-deployment.yaml:4
  current_state: "oc logs -f deployment/vteam-operator -n ambient-code"
  desired_state: "oc logs -f deployment/agentic-operator -n ambient-code"

- gap_id: GAP-BACKEND-README-001
  artifact_path: /vTeam/components/backend/README.md
  gap_type: MISSING_COMPONENT
  severity: HIGH
  description: "Backend component has no README documenting 17 major features"
  source_reference: /vTeam/components/backend/main.go:1
  current_state: null
  desired_state: "Create comprehensive backend README documenting API architecture, GitHub integration, WebSocket messaging, multi-repo support, RFE workflows, etc."

- gap_id: GAP-API-REF-001
  artifact_path: /vTeam/docs/API_REFERENCE.md
  gap_type: MISSING_ENDPOINT
  severity: HIGH
  description: "60 of 64 backend API endpoints are completely undocumented"
  source_reference: /vTeam/components/backend/main.go:96-217
  current_state: null
  desired_state: "Create API reference documentation with all 64 endpoints, request/response schemas, authentication, examples"
```

---

### 4. UpdateRequirement

Represents a specific, actionable change that needs to be made to documentation.

**Fields**:
- `requirement_id` (string, required): Unique identifier (e.g., `REQ-README-001`)
- `artifact_path` (string, required): Path to documentation file to update
- `section` (string, nullable): Specific section within the file (e.g., "Troubleshooting", "Architecture Table")
- `line_range` (string, nullable): Line numbers if applicable (e.g., "81-85")
- `change_type` (enum, required): Type of change needed
  - Values: `CREATE_NEW`, `UPDATE_EXISTING`, `ADD_SECTION`, `FIX_COMMAND`, `FIX_NAME`, `ADD_EXAMPLE`, `ADD_ENDPOINT`, `ADD_VARIABLE`
- `current_content` (string, nullable): Current content that needs to be changed
- `required_content` (string, required): New or updated content to add
- `rationale` (string, required): Why this change is needed
- `source_reference` (string, required): File and line number in source code that justifies this change
- `priority` (enum, required): Priority based on impact
  - Values: `P0_CRITICAL`, `P1_HIGH`, `P2_MEDIUM`, `P3_LOW`
- `estimated_effort` (enum, required): Rough estimate of work required
  - Values: `SMALL` (< 30 min), `MEDIUM` (30 min - 2 hours), `LARGE` (2+ hours), `X_LARGE` (full day+)

**Relationships**:
- `belongs_to` → One DocumentationArtifact
- `addresses` → One or more DocumentationGap

**Validation Rules**:
- `source_reference` must point to actual file in vTeam repository
- If `change_type=UPDATE_EXISTING`, must provide `current_content`
- If `change_type=CREATE_NEW`, `artifact_path` should not exist yet

**Examples**:
```yaml
- requirement_id: REQ-README-001
  artifact_path: /vTeam/README.md
  section: Troubleshooting
  line_range: "81"
  change_type: FIX_COMMAND
  current_content: "oc logs -f deployment/vteam-operator -n ambient-code"
  required_content: "oc logs -f deployment/agentic-operator -n ambient-code"
  rationale: "Actual deployment name is agentic-operator per operator-deployment.yaml"
  source_reference: /vTeam/components/manifests/operator-deployment.yaml:4
  priority: P0_CRITICAL
  estimated_effort: SMALL

- requirement_id: REQ-BACKEND-README-001
  artifact_path: /vTeam/components/backend/README.md
  section: null
  line_range: null
  change_type: CREATE_NEW
  current_content: null
  required_content: |
    # Backend API Service

    Go-based REST API service that provides project management, session orchestration,
    WebSocket messaging, GitHub App integration, and RFE workflow management.

    ## Features
    - 64 HTTP endpoints for project, session, workflow, and admin operations
    - GitHub App OAuth integration with installation tracking
    - Real-time WebSocket messaging with JSONL persistence
    - Multi-repository session support
    - Dual-mode operation (API server or content service sidecar)
    ...
  rationale: "Backend component completely undocumented; 17 major features exist in code"
  source_reference: /vTeam/components/backend/main.go:1
  priority: P0_CRITICAL
  estimated_effort: X_LARGE

- requirement_id: REQ-ENV-VAR-001
  artifact_path: /vTeam/docs/ENVIRONMENT_VARIABLES.md
  section: Runner Variables
  line_range: null
  change_type: ADD_VARIABLE
  current_content: null
  required_content: |
    ### MAIN_REPO_NAME
    - **Type**: String
    - **Required**: No (alternative: MAIN_REPO_INDEX)
    - **Purpose**: Name of the main repository for multi-repo sessions
    - **Used by**: Claude Code Runner
    - **Example**: `myorg/myrepo`
    - **Source**: Injected by operator based on session configuration
  rationale: "MAIN_REPO_NAME is used by wrapper.py:166 but completely undocumented"
  source_reference: /vTeam/components/runners/claude-code-runner/wrapper.py:166
  priority: P1_HIGH
  estimated_effort: SMALL
```

---

## Relationships Diagram

```
DocumentationArtifact
    |
    |-- has_gaps --> DocumentationGap
    |-- requires_updates --> UpdateRequirement
    |-- references --> SourceCodeFile
                           |
                           |-- triggers_gaps --> DocumentationGap

UpdateRequirement
    |
    |-- addresses --> DocumentationGap
    |-- belongs_to --> DocumentationArtifact
```

---

## Validation Contracts

### Contract 1: Documentation Completeness
Every `SourceCodeFile` with `features.length > 0` MUST be `documented_by` at least one `DocumentationArtifact` with `status=CURRENT`.

### Contract 2: Gap Resolution
Every `DocumentationGap` with `severity=HIGH` MUST have at least one `UpdateRequirement` that `addresses` it.

### Contract 3: Accuracy
Every `DocumentationArtifact` with `status=CURRENT` MUST have ZERO `DocumentationGap` records with `severity=HIGH`.

### Contract 4: Source Reference
Every `UpdateRequirement` MUST have a valid `source_reference` that points to an actual file and line number in the vTeam repository.

---

## Usage in Tasks Generation

The `/tasks` command will use this data model to generate ordered tasks:

1. **Query all DocumentationGap records** → Generate audit summary
2. **Group UpdateRequirement by artifact_path** → One task per artifact file
3. **Order by priority** (P0 → P1 → P2 → P3)
4. **Order by dependency** (README before component READMEs; component READMEs before guides)
5. **Mark parallel tasks** (different artifact_path values can be worked on in parallel)

---

**Data Model Complete**: Ready for contract generation and quickstart creation.
