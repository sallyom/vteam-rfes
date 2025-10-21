# UX Team Lead Input: LlamaIndex + pgvectordb RAG Demo

**Feature**: LlamaIndex + pgvectordb demo with end-to-end documentation
**Prepared by**: Uma, UX Team Lead
**Date**: 2025-10-21
**Status**: UX Input for RFE Development

---

## Executive Summary

This demo must deliver an exemplary developer experience. Since the primary deliverable IS the documentation and demo flow itself, UX quality directly determines feature success. This document establishes UX standards, user journey requirements, documentation quality criteria, and measurable success metrics.

**Key UX Principle**: Demos are educational products. Every friction point is a learning barrier that prevents adoption.

---

## 1. UX Quality Standards

### 1.1 Clarity of Instructions

**Standard**: Zero-ambiguity execution path

#### Requirements:
- **UXQ-001**: Each step MUST have a single, unambiguous interpretation
  - Example: "Run `uv run python setup.py demo-dir`" (clear)
  - Anti-pattern: "Set up the environment" (unclear - how?)

- **UXQ-002**: Prerequisites MUST be explicitly stated before any execution steps
  - Location: Top of README, before "Quick Start"
  - Format: Checklist with verification commands
  - Example: "Check Ollama is running: `curl http://localhost:11434`"

- **UXQ-003**: Command outputs MUST include expected success indicators
  - Show users what "good" looks like
  - Example: "You should see: `Server started on port 8080`"

- **UXQ-004**: Directory and path context MUST be explicit in every command block
  - Start each code block with current working directory
  - Example:
    ```
    # In: /path/to/demo-dir
    uv run python start_server.py
    ```

#### Quality Checklist:
- [ ] Can a new developer execute this without asking questions?
- [ ] Are all implicit assumptions made explicit?
- [ ] Does every command show WHERE to run it and WHAT to expect?

### 1.2 Completeness of Examples

**Standard**: End-to-end working examples for every major workflow

#### Requirements:
- **UXQ-005**: MUST provide complete, runnable examples (not fragments)
  - Include all imports, setup, execution, and teardown
  - Examples should be copy-paste executable

- **UXQ-006**: MUST show BOTH happy path AND common variations
  - Default configuration (fastest path to success)
  - Custom configuration examples (learning path)
  - Production-ready examples (deployment path)

- **UXQ-007**: Example data MUST be provided or auto-generated
  - Don't require users to "bring their own" for initial success
  - Include sample documents in the demo
  - Auto-setup scripts should create test data

- **UXQ-008**: MUST demonstrate the complete value chain
  - Document ingestion
  - Query execution
  - Result interpretation
  - End-to-end workflow demonstration

#### Success Criteria:
- User achieves first successful query within 10 minutes
- User understands what they built and why it matters
- User can modify examples to test their own documents

### 1.3 Error Handling and Feedback

**Standard**: Proactive error prevention with clear recovery paths

#### Requirements:
- **UXQ-009**: Common failure points MUST be documented with solutions
  - Create a "Troubleshooting" section
  - List each error message users might encounter
  - Provide exact remediation steps

- **UXQ-010**: Setup scripts MUST validate prerequisites before execution
  - Check for required services (PostgreSQL, Ollama)
  - Verify Python version and dependencies
  - Fail fast with actionable error messages

- **UXQ-011**: Error messages MUST include context and next steps
  - What went wrong (technical detail)
  - Why it matters (impact)
  - How to fix it (action)
  - Example: "PostgreSQL connection failed. The RAG demo requires a running PostgreSQL instance with pgvector extension. Fix: Run `./start-postgres.sh` or see Setup section."

- **UXQ-012**: Provide environment health check command
  - Single command to verify all prerequisites
  - Example: `./check-demo-readiness.sh`
  - Output: Green checkmarks for ready, red X with fix instructions for issues

#### Quality Gates:
- [ ] Have we tested the demo on a fresh machine?
- [ ] Have we documented every error message we encountered?
- [ ] Can users self-diagnose and fix issues without contacting support?

### 1.4 Consistency with Existing Demos

**Standard**: Pattern alignment with llama-stack demo

#### Requirements:
- **UXQ-013**: Directory structure MUST follow established pattern
  ```
  demos/llamaindex-rag/
    README.md              # Main documentation
    setup_*.py            # Setup automation
    client.py             # Direct usage demo
    agent_*.py            # Agent integration demo
    example_data/         # Sample documents
  ```

- **UXQ-014**: README structure MUST match llama-stack pattern
  - Quick Start section (copy-paste commands)
  - Details section (deeper explanation)
  - Setup Details section (prerequisites, architecture)
  - Troubleshooting section (error scenarios)

- **UXQ-015**: Code style and naming MUST be consistent
  - Same CLI argument patterns (`uv run python script.py target-dir`)
  - Same configuration file naming (`*-config.yaml`)
  - Same variable naming conventions

- **UXQ-016**: Documentation tone MUST be consistent
  - Imperative mood for instructions ("Run...", "Create...", "Start...")
  - Present tense for descriptions ("This script creates...")
  - Active voice throughout

#### Design System Compliance:
This follows our demo documentation pattern established in llama-stack. Any deviations must be justified and documented.

---

## 2. User Journey Quality

### 2.1 Making the Happy Path Obvious

**Principle**: The fastest path to success should be the most visible path

#### Journey Design:
- **UXJ-001**: Front-load the Quick Start section
  - Appears first, before architecture or theory
  - Maximum 5 commands from clone to working demo
  - Each command on its own line with clear purpose

- **UXJ-002**: Progressive disclosure of complexity
  - Quick Start: Minimum viable demo (5 min)
  - Customization: Modify for your needs (15 min)
  - Deep Dive: Understanding internals (30+ min)
  - Production: Deployment considerations (reference)

- **UXJ-003**: Visual waypoints for progress
  - Numbered steps (1, 2, 3...)
  - Success indicators ("You should see...")
  - Time estimates ("~2 minutes")
  - Clear section transitions

#### Cognitive Load Reduction:
- **UXJ-004**: Minimize decisions in happy path
  - Provide sensible defaults
  - Make advanced options opt-in
  - Example: Auto-select port, model, database name

- **UXJ-005**: Defer non-critical information
  - Architecture explanations go in "Details" section
  - Alternative approaches in "Advanced" section
  - Theory in separate documentation

### 2.2 Handling Common Mistakes

**Principle**: Design for human error, not perfect execution

#### Error Prevention:
- **UXJ-006**: Idempotent setup scripts
  - Running setup twice should be safe
  - Detect existing installations and offer options
  - Clear cleanup commands for fresh starts

- **UXJ-007**: Guard rails on destructive operations
  - Warn before deleting data or configurations
  - Require explicit confirmation flags
  - Provide rollback instructions

- **UXJ-008**: Path and directory validation
  - Check if user is in correct directory
  - Provide absolute paths in error messages
  - Auto-create missing directories when safe

#### Common Mistake Scenarios:
1. **Running commands in wrong directory**
   - Detection: Scripts check for expected files
   - Recovery: Show current location and expected location

2. **Missing prerequisites**
   - Detection: Validate before execution
   - Recovery: Link to setup instructions for missing component

3. **Port conflicts**
   - Detection: Check port availability before starting services
   - Recovery: Suggest alternative ports or show how to free current port

4. **Stale state from previous runs**
   - Detection: Check for lock files, running processes
   - Recovery: Offer cleanup command

### 2.3 User Feedback at Each Step

**Principle**: Users should always know where they are and what's happening

#### Feedback Requirements:
- **UXJ-009**: Progress indicators for long-running operations
  - Setup scripts show what they're doing
  - Example: "Installing dependencies (2/5)..."
  - Estimated time remaining for operations >30 seconds

- **UXJ-010**: Success confirmations with next steps
  - Explicit "Setup complete!" message
  - Show what was created/configured
  - Provide next command to run

- **UXJ-011**: Real-time feedback during execution
  - Show logs from server startup
  - Display query processing status
  - Indicate when operations complete

- **UXJ-012**: State visibility commands
  - `./status.sh` - Show what's running
  - `./validate.sh` - Check configuration health
  - `./logs.sh` - View recent logs

#### Visual Design for Terminal Output:
```
[SETUP] Installing dependencies...
[SETUP] Configuring database...
[SETUP] Starting services...
[SUCCESS] Demo environment ready!

Next steps:
  1. Start the server: cd demo-dir && ./start.sh
  2. Run the demo: ./demo.sh
```

### 2.4 Reducing Cognitive Load

**Principle**: Minimize mental effort required to understand and execute

#### Strategies:
- **UXJ-013**: One concept per section
  - Don't mix setup with usage with troubleshooting
  - Clear section headers for scanning

- **UXJ-014**: Consistent mental models
  - "Setup" always means one-time configuration
  - "Start" always means launching services
  - "Run" always means executing demos

- **UXJ-015**: Reduced context switching
  - Keep related commands together
  - Minimize directory changes
  - Use absolute paths where possible

- **UXJ-016**: Visual chunking
  - Use code blocks for commands
  - Use tables for comparisons
  - Use lists for sequences
  - Use callouts for important notes

---

## 3. Documentation UX

### 3.1 Visual Hierarchy and Structure

**Principle**: Information architecture supports task completion

#### Structural Requirements:
- **DOCUX-001**: README structure (mandatory order):
  1. One-line description (what this is)
  2. Quick Start (fastest success path)
  3. What You'll Build (outcomes, value)
  4. Prerequisites (detailed requirements)
  5. Usage Examples (variations)
  6. Architecture (how it works)
  7. Troubleshooting (common issues)
  8. Advanced Topics (customization)

- **DOCUX-002**: Heading hierarchy (strictly enforced):
  - H1: Document title only
  - H2: Major sections
  - H3: Subsections
  - H4: Rare, for complex subsections only

- **DOCUX-003**: Visual markers for scanning:
  - Code blocks with language tags
  - Bold for key terms on first use
  - Tables for structured comparisons
  - Lists for sequences or collections

- **DOCUX-004**: Navigation aids:
  - Table of contents for docs >200 lines
  - Cross-references use relative links
  - "Back to top" links for long sections

#### Typography Guidelines:
- Commands: \`code style\`
- File paths: \`code style\`
- Variables: `SCREAMING_SNAKE_CASE` when user-replaceable
- Emphasis: **bold** for importance, *italic* for emphasis

### 3.2 Progressive Disclosure of Complexity

**Principle**: Show simple first, reveal complexity on-demand

#### Implementation:
- **DOCUX-005**: Layered documentation:
  - Level 1: README Quick Start (5 min to success)
  - Level 2: README Details (understand what you built)
  - Level 3: Separate docs for advanced topics
  - Level 4: Code comments for implementation details

- **DOCUX-006**: Expandable sections for optional content:
  ```markdown
  <details>
  <summary>Advanced: Custom configuration options</summary>

  [Complex content here]
  </details>
  ```

- **DOCUX-007**: Separate files for specialized audiences:
  - `README.md` - All users
  - `CONTRIBUTING.md` - Developers extending the demo
  - `ARCHITECTURE.md` - System designers
  - `TROUBLESHOOTING.md` - Support reference

### 3.3 Scanability and Findability

**Principle**: Users should find answers in <10 seconds of scanning

#### Techniques:
- **DOCUX-008**: Front-load key information in sections:
  - First sentence answers "what" or "why"
  - Details follow for those who need them

- **DOCUX-009**: Use parallel structure:
  ```markdown
  ### Option 1: Local Development
  **Use when**: Testing on your laptop
  **Setup time**: 5 minutes
  **Command**: `./setup-local.sh`

  ### Option 2: Cloud Deployment
  **Use when**: Production deployment
  **Setup time**: 15 minutes
  **Command**: `./setup-cloud.sh`
  ```

- **DOCUX-010**: Strategic use of whitespace:
  - Blank lines between concepts
  - Indentation for hierarchical relationships
  - Code blocks separated from prose

- **DOCUX-011**: Search-optimized headers:
  - Use terms users would search for
  - Example: "Error: Port 5432 already in use" (searchable error message)
  - Example: "How to use custom documents" (natural language question)

### 3.4 Action-Oriented Language

**Principle**: Documentation should tell users what to DO, not just what IS

#### Language Requirements:
- **DOCUX-012**: Use imperative mood for instructions:
  - "Run the setup script" (not "The setup script can be run")
  - "Create a directory" (not "A directory should be created")

- **DOCUX-013**: Lead with the action, not the context:
  - Good: "Run `./setup.sh` to configure the demo environment"
  - Bad: "The demo environment can be configured by running `./setup.sh`"

- **DOCUX-014**: Specify outcomes, not just inputs:
  - "Run `./setup.sh` to create a demo-dir with configured services"
  - Not just: "Run `./setup.sh`"

- **DOCUX-015**: Use second person ("you") for engagement:
  - "You can customize the model by editing config.yaml"
  - Avoid: "Users may customize..." or "One can customize..."

---

## 4. Success Metrics

### 4.1 Time to First Success

**Target**: 10 minutes from git clone to successful query

#### Measurement Points:
- **METRIC-001**: Time to complete Quick Start
  - Start: User clones repository
  - End: User sees successful query results
  - Target: 10 minutes (90th percentile)
  - Stretch: 5 minutes (50th percentile)

- **METRIC-002**: Time to understand what was built
  - After first success, user can explain:
    - What components are running
    - How data flows through the system
    - What the demo demonstrates
  - Measured via: Post-demo survey question

#### Instrumentation:
- Add optional telemetry to setup scripts (opt-in)
- Track: start time, completion time, errors encountered
- Anonymized data collection for improvement

### 4.2 Number of Steps Required

**Target**: Maximum 5 commands in happy path

#### Step Inventory:
1. Prerequisites verification (optional automated check)
2. Setup: `uv run python setup.py demo-dir`
3. Start services: `cd demo-dir && ./start.sh`
4. Run demo: `./client.py`
5. Review results: (automatic output)

#### Optimization Goals:
- **METRIC-003**: Reduce setup to single command where possible
- **METRIC-004**: Automate all automatable steps
- **METRIC-005**: Combine steps that have no user decision points

### 4.3 User Confidence Indicators

**Target**: Users feel confident to customize and extend

#### Qualitative Metrics:
- **METRIC-006**: Post-demo capability survey:
  - [ ] "I understand what LlamaIndex RAG does"
  - [ ] "I could modify this to use my own documents"
  - [ ] "I know how to troubleshoot if something breaks"
  - [ ] "I could explain this demo to a colleague"
  - Target: >80% strongly agree

- **METRIC-007**: Documentation completeness:
  - [ ] All commands include expected output
  - [ ] All error messages have remediation steps
  - [ ] All prerequisites have verification commands
  - [ ] All examples are copy-paste runnable
  - Target: 100% checklist completion

#### Behavioral Indicators:
- **METRIC-008**: Customization attempts:
  - Track users who modify demo after initial success
  - Higher rate = higher confidence
  - Measure via: GitHub forks, issues asking "how do I..."

### 4.4 Common Failure Points to Eliminate

**Target**: Zero preventable failures

#### Top Failure Modes (prioritized):
1. **Missing prerequisites** (CRITICAL)
   - Solution: Automated verification script
   - Success: Script catches 100% of missing prereqs

2. **Wrong directory execution** (HIGH)
   - Solution: Scripts check for marker files
   - Success: Scripts fail gracefully with location guidance

3. **Port conflicts** (HIGH)
   - Solution: Scripts check port availability
   - Success: Suggest alternatives or show conflict resolution

4. **Stale state from previous runs** (MEDIUM)
   - Solution: Cleanup script, idempotent setup
   - Success: `./cleanup.sh && ./setup.sh` always works

5. **Version mismatches** (MEDIUM)
   - Solution: Pin all dependency versions
   - Success: Same demo works 6 months later

#### Failure Elimination Strategy:
- **METRIC-009**: Pre-flight checks in setup scripts
- **METRIC-010**: Clear error messages with recovery steps
- **METRIC-011**: Automated cleanup utilities
- **METRIC-012**: Version pinning and compatibility matrix

---

## 5. Customer Considerations

### 5.1 Different Deployment Environments

**Requirement**: Demo must work across diverse environments

#### Environment Matrix:
| Environment | Requirements | Considerations |
|------------|--------------|----------------|
| **Local Development** | Laptop, Docker Desktop | Quick setup, resource constraints |
| **OpenShift Dev Spaces** | Browser-based, cloud resources | Network policies, persistent storage |
| **Cloud VM** | RHEL/Fedora, enterprise security | Firewall rules, SELinux contexts |
| **Air-gapped** | No internet, pre-downloaded artifacts | Dependency vendoring, offline docs |

#### Implementation Requirements:
- **CUST-001**: Environment detection
  - Scripts detect environment type
  - Auto-configure for environment constraints
  - Example: Use different networking for containers vs. host

- **CUST-002**: Alternative installation paths
  - Default: Full automation (requires internet)
  - Alternative: Manual steps documented for air-gapped
  - Alternative: Pre-built container images

- **CUST-003**: Resource requirements documented
  - Minimum: 4GB RAM, 10GB disk
  - Recommended: 8GB RAM, 20GB disk
  - Explain WHY (model size, database size, etc.)

### 5.2 Different Document Types and Sizes

**Requirement**: Demo handles diverse document characteristics

#### Document Type Support:
- **CUST-004**: Sample dataset includes variety:
  - Text: .txt, .md
  - Documents: .pdf, .docx
  - Code: .py, .java, .yaml
  - Structured: .json, .xml

- **CUST-005**: Document size guidance:
  - Small: <100 pages (demo default)
  - Medium: 100-1000 pages (configuration example)
  - Large: >1000 pages (performance considerations documented)

- **CUST-006**: Demonstrate document ingestion:
  - Batch upload script
  - Progress monitoring
  - Error handling for corrupt/unsupported files

#### Customer Use Case Examples:
- Technical documentation (API docs, user manuals)
- Code repositories (search codebase)
- Knowledge bases (internal wikis, SOPs)
- Research papers (academic literature)

### 5.3 Different Skill Levels

**Requirement**: Accessible to beginners, extensible for experts

#### Skill Level Personas:

##### Beginner: "New to RAG"
- **Needs**: Understand what RAG is and why it matters
- **Path**: Quick Start + explanatory comments
- **Success**: Can run demo and explain what happened
- **Support**:
  - **CUST-007**: Glossary of terms
  - **CUST-008**: Architecture diagram with labels
  - **CUST-009**: "What just happened" explanations after each step

##### Intermediate: "Familiar with ML/AI concepts"
- **Needs**: Understand implementation choices
- **Path**: Details section + architecture docs
- **Success**: Can modify demo for their documents
- **Support**:
  - **CUST-010**: Configuration reference
  - **CUST-011**: Performance tuning guide
  - **CUST-012**: Comparison with alternatives

##### Expert: "Building production RAG systems"
- **Needs**: Production readiness checklist
- **Path**: Advanced topics + code review
- **Success**: Can evaluate for production use
- **Support**:
  - **CUST-013**: Security considerations
  - **CUST-014**: Scalability patterns
  - **CUST-015**: Integration examples

### 5.4 Accessibility and Inclusivity

**Requirement**: Demo and documentation accessible to all users

#### Accessibility Standards:
- **CUST-016**: Documentation accessibility:
  - [ ] Plain language (Flesch Reading Ease >60)
  - [ ] Alt text for diagrams and screenshots
  - [ ] Semantic HTML in any web components
  - [ ] Color not sole indicator of meaning

- **CUST-017**: Terminal output accessibility:
  - [ ] Don't rely solely on color (use symbols too)
  - [ ] Screen reader friendly output formatting
  - [ ] Avoid ASCII art that doesn't translate to speech

- **CUST-018**: Inclusive language:
  - [ ] Gender-neutral pronouns
  - [ ] Avoid idioms and colloquialisms
  - [ ] Define technical terms on first use
  - [ ] Avoid ableist language ("sanity check" → "validation check")

#### International Considerations:
- **CUST-019**: English as primary, but:
  - Use simple vocabulary
  - Short sentences (max 20 words)
  - Active voice
  - Consider future translation

- **CUST-020**: Cultural neutrality:
  - Avoid culturally-specific examples
  - Use international date formats (YYYY-MM-DD)
  - Don't assume US-centric infrastructure

---

## 6. Design System Alignment

### 6.1 Demo Documentation Pattern

Following the established pattern from llama-stack demo:

#### File Structure Pattern:
```
demos/llamaindex-rag/
  README.md                    # Main entry point
  setup_llamaindex_rag.py      # Automated setup
  client.py                    # Direct usage example
  agent_example.py             # Agent integration example
  example_documents/           # Sample data
    sample1.pdf
    sample2.txt
    README.md                  # Data attribution
```

#### README Pattern:
1. Title + one-line description
2. Quick Start (command sequence)
3. What This Demonstrates
4. Prerequisites (with verification)
5. Detailed Usage
6. Architecture Overview
7. Troubleshooting
8. Advanced Topics

### 6.2 Consistency Checklist

Before marking as complete, verify:

- [ ] **Naming consistency**: Same naming pattern as llama-stack demo
- [ ] **Command consistency**: Same CLI patterns and flags
- [ ] **Structure consistency**: Same directory layout
- [ ] **Tone consistency**: Same instructional voice
- [ ] **Format consistency**: Same markdown formatting
- [ ] **Code style consistency**: Same Python conventions

---

## 7. Implementation Guidance

### 7.1 Documentation Review Process

This feature requires UX/documentation review at these checkpoints:

1. **Design Review**: Architecture and approach
2. **Draft Review**: Initial README and scripts
3. **Usability Test**: Fresh-eyes testing on new machine
4. **Final Review**: Complete documentation review

### 7.2 Success Criteria Summary

The demo documentation is complete when:

- [ ] A developer with no prior RAG knowledge can complete Quick Start in 10 minutes
- [ ] All commands execute successfully on fresh RHEL 9 VM
- [ ] All error scenarios have documented remediation
- [ ] Documentation passes accessibility review
- [ ] Pattern matches llama-stack demo structure
- [ ] Usability test with 3 developers shows >80% confidence scores

### 7.3 Quality Gates

Must pass before merge:

- [ ] Fresh machine test (clean VM, no prior setup)
- [ ] Documentation review by technical writer
- [ ] UX review for clarity and completeness
- [ ] Accessibility review for inclusive design
- [ ] Consistency review against llama-stack demo

---

## 8. Open Questions for RFE Team

These questions should be answered during RFE development:

1. **Deployment target**: Which environment(s) are priority? Local? OpenShift?
2. **Document corpus**: Do we provide sample docs or use existing corpus?
3. **Vector database**: PostgreSQL+pgvector or separate service?
4. **Model selection**: Which LLM models should demo support?
5. **Performance expectations**: What query latency is acceptable for demo?
6. **Integration scope**: Just LlamaIndex or show integration with other tools?

---

## Appendix: UX Review Rubric

Use this rubric for documentation review:

| Criterion | Weight | Evaluation Questions |
|-----------|--------|---------------------|
| **Clarity** | 25% | Can a new developer execute without asking questions? |
| **Completeness** | 25% | Are all workflows documented end-to-end? |
| **Error Handling** | 20% | Are failure modes documented with remediation? |
| **Consistency** | 15% | Does it match established patterns? |
| **Accessibility** | 15% | Is it inclusive and accessible to all users? |

**Scoring**:
- 5 = Exemplary (publish as template for others)
- 4 = Complete (meets all requirements)
- 3 = Acceptable (minor gaps, document improvements)
- 2 = Needs work (major gaps, requires revision)
- 1 = Incomplete (not ready for review)

**Minimum passing score**: 4.0 average across all criteria

---

## Document History

| Date | Author | Change |
|------|--------|--------|
| 2025-10-21 | Uma (UX Team Lead) | Initial UX input document |

---

**Next Steps**:
1. Product team reviews UX requirements
2. Engineering assesses technical feasibility
3. Technical writer assigned for documentation development
4. UX designer assigned for any visual artifacts (diagrams, screenshots)

**Questions?** This needs to go through design critique once initial draft is ready. Tag @uma for documentation review.
