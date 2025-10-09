
# Implementation Plan: OpenTelemetry Operator Instrumentation for llm-d Tracing

**Branch**: `001-add-opentelemetry-operator` | **Date**: 2025-10-09 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-add-opentelemetry-operator/spec.md`

## Execution Flow (/plan command scope)
```
1. Load feature spec from Input path
   → If not found: ERROR "No feature spec at {path}"
2. Fill Technical Context (scan for NEEDS CLARIFICATION)
   → Detect Project Type from file system structure or context (web=frontend+backend, mobile=app+api)
   → Set Structure Decision based on project type
3. Fill the Constitution Check section based on the content of the constitution document.
4. Evaluate Constitution Check section below
   → If violations exist: Document in Complexity Tracking
   → If no justification possible: ERROR "Simplify approach first"
   → Update Progress Tracking: Initial Constitution Check
5. Execute Phase 0 → research.md
   → If NEEDS CLARIFICATION remain: ERROR "Resolve unknowns"
6. Execute Phase 1 → contracts, data-model.md, quickstart.md, agent-specific template file (e.g., `CLAUDE.md` for Claude Code, `.github/copilot-instructions.md` for GitHub Copilot, `GEMINI.md` for Gemini CLI, `QWEN.md` for Qwen Code or `AGENTS.md` for opencode).
7. Re-evaluate Constitution Check section
   → If new violations: Refactor design, return to Phase 1
   → Update Progress Tracking: Post-Design Constitution Check
8. Plan Phase 2 → Describe task generation approach (DO NOT create tasks.md)
9. STOP - Ready for /tasks command
```

**IMPORTANT**: The /plan command STOPS at step 7. Phases 2-4 are executed by other commands:
- Phase 2: /tasks command creates tasks.md
- Phase 3-4: Implementation execution (manual or via tools)

## Summary
Deploy OpenTelemetry Operator auto-instrumentation alongside Jaeger to provide end-to-end distributed tracing for llm-d inference scheduler. The solution will automatically instrument vLLM pods using Instrumentation CRs, integrate with vLLM's native `--otlp-traces-endpoint` tracing, and enable trace context propagation from client requests through Envoy gateway, EPP routing, to vLLM model serving. This provides llm-d operators with unified visibility into request latency, routing decisions, and model inference performance without requiring manual code changes.

## Technical Context
**Language/Version**: Python 3.11+ (vLLM), YAML (Kubernetes manifests)
**Primary Dependencies**: OpenTelemetry Operator, Jaeger Operator, vLLM (with native OTel support), Kubernetes 1.27+, Envoy Gateway API
**Storage**: Jaeger backend (in-memory for dev, Elasticsearch/Cassandra/S3 for production), trace data via OTLP
**Testing**: kubectl dry-run validation, contract tests for OTLP endpoints, integration tests for trace propagation, performance tests (<5ms p99 overhead)
**Target Platform**: Kubernetes clusters (Linux amd64/arm64), llm-d inference scheduler components
**Project Type**: Infrastructure/Kubernetes manifests - Kustomize-based component structure
**Performance Goals**: <5ms p99 latency overhead from tracing instrumentation, support for high-throughput inference workloads
**Constraints**: Zero code changes to vLLM containers, auto-injection via OpenTelemetry Operator, W3C Trace Context propagation, minimal resource overhead
**Scale/Scope**: Multi-tenant llm-d clusters, distributed traces across Envoy → EPP → vLLM, production-grade observability backend

## Constitution Check
*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Status**: PASS - Infrastructure project using declarative manifests

This feature involves Kubernetes infrastructure deployment (OpenTelemetry Operator, Instrumentation CRs, Jaeger) rather than application code. Key compliance points:
- ✅ **Simplicity**: Uses upstream operators (OTel, Jaeger) rather than custom controllers
- ✅ **Zero Code Changes**: Auto-instrumentation via Instrumentation CRs - no application modifications
- ✅ **Kubernetes-Native**: Declarative YAML manifests following llm-d's Kustomize component structure
- ✅ **Testing**: Contract tests for OTLP schemas, integration tests for trace propagation, performance validation
- ✅ **Documentation**: Deployment guides, troubleshooting runbooks, example traces

**Note**: Constitution template is not yet ratified for this repository. Check applies general best practices for infrastructure-as-code projects.

## Project Structure

### Documentation (this feature)
```
specs/[###-feature]/
├── plan.md              # This file (/plan command output)
├── research.md          # Phase 0 output (/plan command)
├── data-model.md        # Phase 1 output (/plan command)
├── quickstart.md        # Phase 1 output (/plan command)
├── contracts/           # Phase 1 output (/plan command)
└── tasks.md             # Phase 2 output (/tasks command - NOT created by /plan)
```

### Source Code (llm-d-inference-scheduler repository)
```
deploy/
├── components/
│   ├── opentelemetry-operator/          # New component for OTel Operator
│   │   ├── kustomization.yaml
│   │   ├── operator-deployment.yaml
│   │   └── instrumentation-cr.yaml      # Instrumentation custom resource
│   ├── jaeger/                           # New component for Jaeger backend
│   │   ├── kustomization.yaml
│   │   ├── jaeger-operator.yaml
│   │   └── jaeger-instance.yaml
│   └── vllm-sim/                         # Existing component - will be updated
│       ├── kustomization.yaml            # Update to include OTel config
│       └── deployment.yaml               # Update with --otlp-traces-endpoint
├── config/
│   └── tracing/                          # New config directory
│       ├── sampling-config.yaml
│       └── otlp-endpoints.yaml
└── environments/
    ├── dev/
    │   └── kustomization.yaml            # Include tracing components
    └── prod/
        └── kustomization.yaml            # Production tracing config

docs/
├── tracing/                              # New documentation
│   ├── deployment-guide.md
│   ├── troubleshooting.md
│   └── architecture.md
└── examples/
    └── tracing/
        ├── trace-queries.md
        └── sample-traces.yaml

tests/
└── integration/
    └── tracing/                          # New test suite
        ├── test_trace_propagation.sh
        ├── test_jaeger_deployment.sh
        └── test_instrumentation_injection.sh
```

**Structure Decision**: Kubernetes infrastructure using Kustomize components. This feature adds two new components (`opentelemetry-operator/`, `jaeger/`) to the existing `deploy/components/` directory and updates the `vllm-sim` component to enable vLLM's native OTLP tracing. Configuration follows the existing pattern of environment-specific overlays (dev/prod) with base manifests in `components/`.

## Phase 0: Outline & Research
1. **Extract unknowns from Technical Context** above:
   - For each NEEDS CLARIFICATION → research task
   - For each dependency → best practices task
   - For each integration → patterns task

2. **Generate and dispatch research agents**:
   ```
   For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"
   ```

3. **Consolidate findings** in `research.md` using format:
   - Decision: [what was chosen]
   - Rationale: [why chosen]
   - Alternatives considered: [what else evaluated]

**Output**: research.md with all NEEDS CLARIFICATION resolved

## Phase 1: Design & Contracts
*Prerequisites: research.md complete*

1. **Extract entities from feature spec** → `data-model.md`:
   - Entity name, fields, relationships
   - Validation rules from requirements
   - State transitions if applicable

2. **Generate API contracts** from functional requirements:
   - For each user action → endpoint
   - Use standard REST/GraphQL patterns
   - Output OpenAPI/GraphQL schema to `/contracts/`

3. **Generate contract tests** from contracts:
   - One test file per endpoint
   - Assert request/response schemas
   - Tests must fail (no implementation yet)

4. **Extract test scenarios** from user stories:
   - Each story → integration test scenario
   - Quickstart test = story validation steps

5. **Update agent file incrementally** (O(1) operation):
   - Run `.specify/scripts/bash/update-agent-context.sh cursor`
     **IMPORTANT**: Execute it exactly as specified above. Do not add or remove any arguments.
   - If exists: Add only NEW tech from current plan
   - Preserve manual additions between markers
   - Update recent changes (keep last 3)
   - Keep under 150 lines for token efficiency
   - Output to repository root

**Output**: data-model.md, /contracts/*, failing tests, quickstart.md, agent-specific file

## Phase 2: Task Planning Approach
*This section describes what the /tasks command will do - DO NOT execute during /plan*

**Task Generation Strategy**:
- Load `.specify/templates/tasks-template.md` as base
- Generate tasks from Phase 1 design docs (contracts, data model, quickstart)
- Infrastructure tasks organized by component deployment order
- Contract validation tests before implementation
- Integration tests after component deployment
- Documentation tasks at the end

**Ordering Strategy**:
1. **Foundation**: Namespace, RBAC, prerequisites
2. **Backend Components**: Jaeger Operator → Jaeger instance → OpenTelemetry Operator
3. **CRD Extension**: InferencePool CRD schema updates, validation webhooks
4. **Integration**: Instrumentation CR, vLLM deployment updates
5. **Testing**: Contract tests [P], integration tests, performance validation
6. **Documentation**: Deployment guides, troubleshooting, examples

**Parallel Execution Opportunities** (marked [P]):
- Jaeger and OpenTelemetry Operator deployments (independent)
- Contract test files (independent validation)
- Documentation files (independent writing)
- Environment-specific Kustomize overlays (dev/prod)

**Estimated Task Breakdown**:
- Infrastructure setup: 8-10 tasks
- CRD and controller changes: 5-7 tasks
- Configuration and manifests: 10-12 tasks
- Testing: 8-10 tasks (contract + integration + performance)
- Documentation: 5-7 tasks
- **Total**: 36-46 numbered, ordered tasks in tasks.md

**Key Dependencies**:
- Jaeger must be deployed before creating Instrumentation CR (needs OTLP endpoint)
- OpenTelemetry Operator must be installed before creating Instrumentation CR
- InferencePool CRD must be updated before vLLM deployments can use tracing config
- Contract tests must pass before integration testing
- Integration tests must pass before quickstart validation

**IMPORTANT**: This phase is executed by the /tasks command, NOT by /plan

## Phase 3+: Future Implementation
*These phases are beyond the scope of the /plan command*

**Phase 3**: Task execution (/tasks command creates tasks.md)  
**Phase 4**: Implementation (execute tasks.md following constitutional principles)  
**Phase 5**: Validation (run tests, execute quickstart.md, performance validation)

## Complexity Tracking
*Fill ONLY if Constitution Check has violations that must be justified*

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |


## Progress Tracking
*This checklist is updated during execution flow*

**Phase Status**:
- [x] Phase 0: Research complete (/plan command) - ✅ research.md created
- [x] Phase 1: Design complete (/plan command) - ✅ data-model.md, contracts/, quickstart.md created
- [x] Phase 2: Task planning complete (/plan command - describe approach only) - ✅ Approach documented
- [ ] Phase 3: Tasks generated (/tasks command) - NOT YET EXECUTED
- [ ] Phase 4: Implementation complete
- [ ] Phase 5: Validation passed

**Gate Status**:
- [x] Initial Constitution Check: PASS - Infrastructure project using upstream operators
- [x] Post-Design Constitution Check: PASS - Design follows Kubernetes-native patterns
- [x] All NEEDS CLARIFICATION resolved - Research phase addressed all unknowns
- [x] Complexity deviations documented - N/A - No constitutional violations

---
*Based on Constitution v2.1.1 - See `/memory/constitution.md`*
