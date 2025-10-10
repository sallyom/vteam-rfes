# Data Model: vTeam Documentation Entities

**Feature**: Update README and Other Docs for vTeam
**Date**: 2025-10-10
**Status**: Complete

## Overview

This data model defines the structure of documentation entities and their relationships. Since this is a documentation feature, "entities" refer to documentation artifacts (files, sections, code blocks) rather than traditional data models.

---

## Entity 1: DocumentationFile

**Description**: Represents a markdown documentation file in the vTeam repository.

### Fields

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `path` | string | Yes | Valid file path in vTeam repo | Absolute or relative path to markdown file |
| `filename` | string | Yes | `*.md` pattern | Name of the documentation file |
| `type` | enum | Yes | `overview`, `guide`, `reference` | Classification of documentation |
| `sections` | Section[] | Yes | Min 1 section | Ordered list of documentation sections |
| `crossReferences` | Reference[] | No | Valid paths | Links to other documentation files |
| `status` | enum | Yes | `exists`, `needs_update`, `needs_creation` | Current state of the file |

### Relationships
- **Has many** Section (one-to-many)
- **References** other DocumentationFile instances (many-to-many via crossReferences)

### State Transitions
```
needs_creation → exists → needs_update → updated
```

### Validation Rules
1. All `crossReferences` must point to valid markdown files or external URLs
2. Sections must have unique `id` values within the file
3. Files must follow GitHub Flavored Markdown syntax
4. Code blocks must specify language for syntax highlighting

---

## Entity 2: Section

**Description**: Represents a logical section within a documentation file (heading + content).

### Fields

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `id` | string | Yes | Unique within file | Auto-generated from heading text |
| `heading` | string | Yes | Non-empty | Markdown heading text (without `#` symbols) |
| `level` | integer | Yes | 1-6 | Heading level (H1=1, H2=2, etc.) |
| `content` | string | Yes | Valid markdown | Section body content |
| `codeBlocks` | CodeBlock[] | No | - | Embedded code examples |
| `requirements` | string[] | No | Valid FR-XXX IDs | Functional requirements addressed by this section |

### Relationships
- **Belongs to** DocumentationFile (many-to-one)
- **Has many** CodeBlock (one-to-many)

### Validation Rules
1. Heading text must be concise (< 100 characters)
2. Content must be non-empty unless section contains sub-sections
3. Code blocks must be properly fenced with language identifiers
4. Requirements must reference valid functional requirements from spec.md

---

## Entity 3: CodeBlock

**Description**: Represents a code example or command snippet within documentation.

### Fields

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `language` | string | Yes | Valid GFM language | Syntax highlighting language (bash, yaml, typescript, go, etc.) |
| `code` | string | Yes | Non-empty | Code content |
| `description` | string | No | - | Optional description or context for the code block |
| `testable` | boolean | Yes | Default: false | Whether this code block should be executable/testable |
| `expectedOutput` | string | No | - | Expected output for testable code blocks |

### Relationships
- **Belongs to** Section (many-to-one)

### Validation Rules
1. Language must be valid for GitHub Flavored Markdown syntax highlighting
2. Code must be properly formatted and syntactically valid for its language
3. If `testable=true`, must include verification strategy
4. Bash commands must use consistent namespace (ambient-code)

---

## Entity 4: Reference

**Description**: Represents a cross-reference between documentation files or to external resources.

### Fields

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `sourceFile` | string | Yes | Valid path | File containing the reference |
| `targetPath` | string | Yes | Valid path or URL | Target file or external URL |
| `linkText` | string | Yes | Non-empty | Display text for the link |
| `referenceType` | enum | Yes | `internal`, `external`, `anchor` | Type of reference |
| `verified` | boolean | Yes | Default: false | Whether link has been validated |

### Relationships
- **References** DocumentationFile (many-to-one for internal references)

### Validation Rules
1. Internal references must use relative paths from source file
2. External references must be valid HTTP/HTTPS URLs
3. Anchor references must point to existing section IDs
4. All references must be verified before documentation is considered complete

---

## Entity 5: DeploymentScript

**Description**: Represents the deploy.sh script and its configuration options (referenced by documentation).

### Fields

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `path` | string | Yes | `components/manifests/deploy.sh` | Script location |
| `options` | ScriptOption[] | Yes | Min 1 option | Available command-line options |
| `environmentVariables` | EnvVar[] | No | - | Environment variables that affect behavior |
| `dependencies` | string[] | Yes | Valid commands | Required CLI tools (oc, kustomize, etc.) |
| `defaultNamespace` | string | Yes | `ambient-code` | Default deployment namespace |

### Relationships
- **Documented by** multiple DocumentationFile instances
- **Requires** prerequisites (CLI tools, cluster access)

### Validation Rules
1. Documentation must accurately reflect script's actual capabilities
2. All options must be tested and verified
3. Dependencies must be listed in prerequisites section
4. Environment variables must specify expected format and defaults

---

## Entity 6: ComponentArchitecture

**Description**: Represents architectural information about vTeam components (referenced by documentation).

### Fields

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `componentName` | string | Yes | Non-empty | Name of the component (Frontend, Backend, Operator, Runner) |
| `technology` | string | Yes | Non-empty | Primary technology/framework (Next.js 15, Go 1.24, etc.) |
| `language` | string | Yes | Non-empty | Programming language |
| `dependencies` | string[] | Yes | Non-empty | Key dependencies and versions |
| `sourcePath` | string | Yes | Valid directory | Path to component source code |
| `description` | string | Yes | 1-3 sentences | Brief component description |

### Relationships
- **Documented in** README.md architecture section
- **Has** source code in repository

### Validation Rules
1. Technology and version must match package.json or go.mod
2. Dependencies must reflect actual codebase dependencies
3. Description must be accurate and concise
4. Source path must be valid relative to repository root

---

## Derived Documentation Structure

Based on the data model, documentation files map to entities as follows:

### README.md (Root-level overview)
- **Sections**:
  - Overview (description of vTeam platform)
  - Architecture (ComponentArchitecture entities → table)
  - Quick Start (references to getting-started.md)
  - Deployment (references to OPENSHIFT_DEPLOY.md)
  - Git Authentication (references to GIT_AUTH_SETUP.md)

- **Requirements Addressed**: FR-001, FR-002, FR-003, FR-004

### components/README.md (Component overview)
- **Sections**:
  - Directory Structure (component paths)
  - Architecture Flow (7-step agentic session workflow)
  - Quick Start (production, local dev, custom images)

- **Requirements Addressed**: FR-005, FR-006, FR-007

### docs/getting-started.md (Comprehensive guide)
- **Sections**:
  - Prerequisites (tools, API keys)
  - Local Development Setup (CRC installation)
  - Production Deployment (login → deploy → verify)
  - Git Authentication (secret creation commands)
  - Post-Deployment (project creation, runner secrets)
  - Troubleshooting (common issues + solutions)

- **Requirements Addressed**: FR-008, FR-009, FR-010, FR-011, FR-012, FR-013

### docs/OPENSHIFT_DEPLOY.md (Deployment guide)
- **Sections**:
  - Prerequisites (oc, kustomize, registry)
  - Deployment Script Options (DeploymentScript entity details)
  - Git Authentication Setup (link to GIT_AUTH_SETUP.md)
  - Post-Deployment Setup (runner secrets, projects)
  - Cleanup (uninstall commands)

- **Requirements Addressed**: FR-014, FR-015, FR-016, FR-017

### Cross-Cutting Requirements
- **Consistency** (FR-018, FR-019): Enforced via namespace and registry validation
- **Path Validation** (FR-020): Enforced via Reference entity validation
- **Code Formatting** (FR-021): Enforced via CodeBlock entity validation
- **Self-Contained** (FR-022): Enforced via Section completeness validation
- **Progressive Disclosure** (FR-023): Enforced via documentation hierarchy

---

## Validation Checklist

Before documentation is considered complete, validate:

- [ ] All DocumentationFile entities have `status=updated`
- [ ] All Section entities have non-empty content
- [ ] All CodeBlock entities use valid language identifiers
- [ ] All Reference entities have `verified=true`
- [ ] ComponentArchitecture entities match actual source code
- [ ] DeploymentScript documentation matches actual script capabilities
- [ ] Cross-references form a consistent navigation graph
- [ ] No broken links or invalid paths
- [ ] All functional requirements (FR-001 through FR-023) are addressed

---

**Generated**: 2025-10-10 by /plan command execution (Phase 1)
