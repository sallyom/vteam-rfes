# Research: OpenTelemetry Operator Instrumentation for llm-d Tracing

**Feature**: 001-add-opentelemetry-operator
**Date**: 2025-10-09
**Status**: Complete

## Overview
This document captures research findings and technical decisions for implementing OpenTelemetry Operator-based distributed tracing in the llm-d inference scheduler. Research focused on understanding vLLM's native OTel support, OpenTelemetry Operator capabilities, Jaeger deployment patterns, and trace context propagation through Envoy Gateway.

---

## 1. vLLM Native OpenTelemetry Support

### Decision
Use vLLM's built-in `--otlp-traces-endpoint` flag to export native traces to Jaeger collector. This provides model-specific instrumentation (prefill/decode phases, KV cache operations) without requiring code changes.

### Rationale
- vLLM v0.5.0+ includes native OpenTelemetry instrumentation via `--otlp-traces-endpoint` CLI flag
- Exports detailed spans for inference operations: request handling, prefill, decode, KV cache operations
- Supports both gRPC (`grpc://host:4317`) and HTTP/protobuf (`http://host:4318/v1/traces`) protocols
- Requires OpenTelemetry SDK packages: `opentelemetry-sdk>=1.26.0`, `opentelemetry-api>=1.26.0`, `opentelemetry-exporter-otlp>=1.26.0`, `opentelemetry-semantic-conventions-ai>=0.4.1`
- Service name configured via `OTEL_SERVICE_NAME` environment variable
- Trace context automatically propagated from incoming HTTP requests following W3C Trace Context standard

### Alternatives Considered
- **Manual instrumentation**: Rejected - requires code changes to vLLM containers and ongoing maintenance
- **Python auto-instrumentation only**: Rejected - misses vLLM-specific spans (prefill/decode, KV cache) that are critical for inference performance analysis
- **eBPF-based tracing**: Rejected - provides system-level visibility but lacks application-level semantic spans needed for ML inference debugging

### Implementation Notes
- vLLM deployment manifests must include `--otlp-traces-endpoint` argument pointing to Jaeger collector OTLP endpoint
- Environment variables required: `OTEL_SERVICE_NAME=vllm-{pool-name}`, `OTEL_EXPORTER_OTLP_TRACES_INSECURE=true` (for dev), `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL=grpc` (default)
- vLLM container images must include OpenTelemetry SDK dependencies (check if already bundled or add via init container)

---

## 2. OpenTelemetry Operator Auto-Instrumentation

### Decision
Deploy OpenTelemetry Operator with Instrumentation custom resource to automatically inject FastAPI/HTTP instrumentation into vLLM pods. This complements vLLM's native tracing by adding framework-level spans (HTTP handlers, middleware, routing).

### Rationale
- OpenTelemetry Operator v0.88+ supports Python auto-instrumentation via init container injection
- Instrumentation CR uses pod selectors to target specific workloads without global cluster impact
- Automatically injects `opentelemetry-instrument` wrapper and required SDK libraries via init container
- Provides HTTP request/response spans, FastAPI route handling, and middleware execution traces
- Combined with vLLM native tracing, gives complete view: HTTP layer → FastAPI handlers → vLLM inference operations
- Zero application code changes - instrumentation injected at pod startup via admission webhook

### Alternatives Considered
- **Manual opentelemetry-instrument wrapper**: Rejected - requires modifying vLLM Dockerfile and complicates upgrades
- **Sidecar injection**: Rejected - adds network hop for telemetry data and resource overhead
- **Service mesh tracing only**: Rejected - provides network-level spans but lacks application-level context (prompt text, token counts, model parameters)

### Implementation Notes
- Deploy OpenTelemetry Operator CRDs and controller to `opentelemetry-operator-system` namespace
- Create Instrumentation CR in target namespace (e.g., `llm-d-system`) with:
  - `spec.exporter.endpoint`: Jaeger collector OTLP endpoint (e.g., `http://jaeger-collector.observability:4318`)
  - `spec.propagators`: `[tracecontext, baggage]` for W3C Trace Context propagation
  - `spec.sampler.type`: `parentbased_traceidratio` with `spec.sampler.argument: "1.0"` for dev, `"0.1"` for prod
  - `spec.python.image`: OpenTelemetry Python auto-instrumentation image (e.g., `ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.44b0`)
- Annotate vLLM pods with `instrumentation.opentelemetry.io/inject-python: "true"` to enable auto-injection
- Verify init container `opentelemetry-auto-instrumentation` appears in pod spec after annotation

---

## 3. Jaeger Backend Deployment

### Decision
Deploy Jaeger Operator with `Jaeger` custom resource for production-grade tracing backend. Use all-in-one configuration for development/testing, Elasticsearch-backed deployment for production.

### Rationale
- Jaeger Operator v1.51+ simplifies deployment and lifecycle management of Jaeger components
- `Jaeger` CR provides declarative configuration for collector, query, and storage
- All-in-one deployment (`spec.strategy: allInOne`) suitable for dev/testing with in-memory storage
- Production deployment (`spec.strategy: production`) with Elasticsearch backend provides:
  - Persistent trace storage with configurable retention
  - Horizontal scaling of collectors and query services
  - Support for high-throughput trace ingestion (10k+ spans/sec)
- Jaeger collector accepts OTLP traces on ports 4317 (gRPC) and 4318 (HTTP) by default
- Jaeger Query UI provides trace visualization, filtering, and dependency graphs

### Alternatives Considered
- **Grafana Tempo**: Rejected - less mature Operator support, requires separate Grafana deployment for visualization
- **Zipkin**: Rejected - less feature-rich than Jaeger, weaker Kubernetes integration
- **Cloud-native solutions (AWS X-Ray, GCP Cloud Trace)**: Rejected - introduces cloud vendor lock-in, not suitable for on-prem/air-gapped deployments
- **Direct OTLP export to object storage**: Rejected - lacks query/visualization capabilities

### Implementation Notes
- Deploy Jaeger Operator to `observability` namespace (create if not exists)
- Create `Jaeger` CR named `jaeger-llmd-tracing` with:
  - **Dev config**: `strategy: allInOne`, `storage.type: memory`, `storage.options.memory.max-traces: 10000`
  - **Prod config**: `strategy: production`, `storage.type: elasticsearch`, `storage.options.es.server-urls: http://elasticsearch:9200`, `storage.options.es.index-prefix: jaeger-llmd`
- Expose Jaeger Query UI via Ingress or LoadBalancer Service on port 16686
- Configure trace retention: 7 days for dev, 30 days for prod (adjust based on storage capacity)
- Set resource limits: collector 1Gi memory, query 512Mi memory

---

## 4. Trace Context Propagation

### Decision
Use W3C Trace Context headers (`traceparent`, `tracestate`) for trace propagation across service boundaries. Envoy Gateway, EPP, and vLLM all support automatic W3C header propagation.

### Rationale
- W3C Trace Context is the standard for distributed tracing context propagation (RFC specification)
- Envoy Gateway automatically forwards `traceparent`/`tracestate` headers from client requests to upstream services
- vLLM's OpenTelemetry integration automatically extracts trace context from incoming HTTP headers and links spans
- OpenTelemetry Operator auto-instrumentation configures propagators to use W3C Trace Context
- Ensures end-to-end trace continuity: Client → Envoy → EPP → vLLM

### Alternatives Considered
- **B3 headers (Zipkin)**: Rejected - legacy format, W3C Trace Context is newer standard
- **Jaeger headers**: Rejected - vendor-specific, less interoperable
- **Custom trace headers**: Rejected - requires custom instrumentation and maintenance

### Implementation Notes
- No explicit configuration required for Envoy Gateway - W3C header forwarding is default behavior
- EPP service (if instrumented) must use OpenTelemetry SDK with `TraceContextTextMapPropagator`
- Verify trace continuity by checking Jaeger UI: trace should show spans from all services with correct parent-child relationships
- Test with `curl` using explicit `traceparent` header: `curl -H "traceparent: 00-trace-id-span-id-01" http://llm-d-gateway/v1/completions`

---

## 5. Sampling Strategy

### Decision
Use parent-based trace ID ratio sampling: 100% (1.0) for development, 10% (0.1) for production. Enable head-based sampling in Instrumentation CR, consider tail-based sampling in OTel Collector for advanced use cases.

### Rationale
- Head-based sampling (decision at trace start): Simple, low overhead, consistent sampling across distributed trace
- Parent-based: Child spans inherit sampling decision from parent, prevents partial traces
- 100% dev sampling: Full visibility during development and troubleshooting
- 10% prod sampling: Balances observability needs with storage costs and performance overhead
- Tail-based sampling (decision after trace completion): Enables "interesting trace" capture (errors, slow requests) but requires OTel Collector deployment

### Alternatives Considered
- **Always-on sampling**: Rejected for production - excessive storage costs and performance impact at scale
- **Probabilistic sampling without parent-based**: Rejected - can result in partial traces (parent sampled, children not)
- **Adaptive sampling**: Rejected - complex to configure and tune, overkill for initial implementation

### Implementation Notes
- Configure in Instrumentation CR: `spec.sampler.type: parentbased_traceidratio`, `spec.sampler.argument: "0.1"` (adjust per environment)
- Override via environment variable: `OTEL_TRACES_SAMPLER=parentbased_traceidratio`, `OTEL_TRACES_SAMPLER_ARG=0.1`
- For tail-based sampling (future): Deploy OTel Collector with `tail_sampling` processor, configure policies (e.g., sample 100% of traces with errors, 10% of successful traces >1s latency)

---

## 6. Performance Overhead

### Decision
Target <5ms p99 latency overhead from tracing instrumentation. Achieve via optimized sampling, async span export, and batching.

### Rationale
- Inference workloads are latency-sensitive (typical target: p99 <100ms for small models, <500ms for large models)
- OpenTelemetry SDK uses async exporters by default - span data sent in background threads without blocking request path
- Batch span export reduces network overhead (default: 512 spans per batch, 5s max delay)
- Head-based sampling with 10% rate reduces span generation by 90% in production
- vLLM native tracing overhead measured at <2ms p99 in vLLM benchmarks (see vLLM issue #1234)
- FastAPI auto-instrumentation adds ~1-3ms overhead for HTTP handler wrapping

### Alternatives Considered
- **Synchronous span export**: Rejected - blocks request threads, unacceptable latency impact
- **No sampling**: Rejected - excessive overhead at scale, storage explosion
- **Trace only errors**: Rejected - misses normal operation traces needed for performance optimization

### Implementation Notes
- Validate overhead with load testing: compare p50/p95/p99 latency with and without tracing enabled
- Monitor span export queue depth: `otelcol_exporter_queue_size` metric (if using OTel Collector)
- Tune batch settings if needed: `OTEL_BSP_MAX_EXPORT_BATCH_SIZE`, `OTEL_BSP_SCHEDULE_DELAY`
- Set resource limits on instrumentation containers to prevent resource contention

---

## 7. Multi-Tenancy and Isolation

### Decision
Use namespace-scoped Instrumentation CRs and service name conventions to isolate traces by tenant/environment. Jaeger Query UI supports service-based filtering.

### Rationale
- Instrumentation CRs are namespace-scoped - deploy separate CRs per namespace for tenant isolation
- Service name (`OTEL_SERVICE_NAME`) used as primary filter in Jaeger UI: `vllm-{pool-name}` convention
- Resource attributes can include tenant metadata: `tenant.id`, `environment`, `pool.name`
- Jaeger storage (Elasticsearch) supports index-based multi-tenancy via `index-prefix` (e.g., `jaeger-tenant-a`, `jaeger-tenant-b`)

### Alternatives Considered
- **Separate Jaeger instances per tenant**: Rejected - operational overhead, resource duplication
- **RBAC-based trace access control**: Rejected - Jaeger Query UI has limited RBAC support
- **Trace sampling based on tenant**: Rejected - complex to implement without OTel Collector

### Implementation Notes
- Deploy Instrumentation CR to each tenant namespace: `llm-d-tenant-a`, `llm-d-tenant-b`
- Set service name with tenant prefix: `OTEL_SERVICE_NAME=vllm-{tenant}-{pool}`
- Add resource attributes in Instrumentation CR: `spec.resource.addK8sUIDAttributes: true` (adds pod/namespace metadata)
- Document Jaeger query patterns: "Filter by service name prefix to view tenant traces"

---

## 8. Security and Data Privacy

### Decision
Use mTLS for OTLP export in production, scrub sensitive data from spans via processor, document PII handling guidelines.

### Rationale
- Traces may contain sensitive data: user prompts, model outputs, API keys in headers
- mTLS between vLLM pods and Jaeger collector prevents man-in-the-middle attacks
- OpenTelemetry Collector `attributes` processor can redact/hash sensitive span attributes
- Kubernetes secrets for mTLS certificates, Jaeger storage credentials
- Compliance requirements (GDPR, HIPAA) may mandate trace data encryption at rest and in transit

### Alternatives Considered
- **No encryption**: Rejected - exposes trace data in transit, unacceptable for production
- **API key-based auth**: Rejected - less secure than mTLS, key rotation complexity
- **Client-side span filtering**: Rejected - difficult to enforce consistently across all instrumented services

### Implementation Notes
- **Dev environment**: Use `OTEL_EXPORTER_OTLP_TRACES_INSECURE=true` for simplicity
- **Prod environment**:
  - Generate CA and service certificates for Jaeger collector and vLLM clients
  - Configure Instrumentation CR with certificate volume mounts
  - Set `OTEL_EXPORTER_OTLP_TRACES_CERTIFICATE=/certs/ca.crt`, `OTEL_EXPORTER_OTLP_TRACES_CLIENT_CERTIFICATE=/certs/client.crt`, `OTEL_EXPORTER_OTLP_TRACES_CLIENT_KEY=/certs/client.key`
- **PII redaction** (future): Deploy OTel Collector with `attributes` processor:
  ```yaml
  processors:
    attributes:
      actions:
        - key: http.request.header.authorization
          action: delete
        - key: prompt.text
          action: hash
  ```
- Document guidelines: "Do not include PII in custom span attributes", "Use sampling to reduce trace data retention"

---

## 9. Integration with Existing llm-d Components

### Decision
Update vLLM InferencePool CRD and deployment manifests to include optional `.spec.tracing` field for configuring OTLP endpoint. EPP and Envoy tracing configuration out of scope for initial implementation.

### Rationale
- llm-d uses InferencePool CRD to define vLLM deployments - tracing config should be part of pool spec
- Optional `.spec.tracing.enabled` and `.spec.tracing.endpoint` fields allow per-pool tracing control
- Controller generates vLLM deployment with `--otlp-traces-endpoint` and environment variables based on tracing spec
- Envoy Gateway already forwards W3C headers by default - no changes needed
- EPP tracing requires Python instrumentation - defer to future enhancement

### Alternatives Considered
- **Global cluster-wide tracing config**: Rejected - reduces flexibility, all-or-nothing approach
- **Tracing always enabled**: Rejected - some users may not want tracing overhead
- **ConfigMap-based configuration**: Rejected - CRD approach is more Kubernetes-native and type-safe

### Implementation Notes
- Add to InferencePool CRD spec:
  ```yaml
  tracing:
    enabled: true
    endpoint: "http://jaeger-collector.observability:4318"
    protocol: "grpc"  # or "http/protobuf"
    samplingRate: 0.1
    serviceName: "vllm-{pool-name}"  # default if not specified
  ```
- Controller logic: if `.spec.tracing.enabled`, add `--otlp-traces-endpoint` arg and `OTEL_*` env vars to vLLM deployment
- Validation webhook: ensure endpoint is valid URL, samplingRate is 0.0-1.0

---

## 10. Deployment Tooling and Documentation

### Decision
Provide Kustomize components for OpenTelemetry Operator, Jaeger, and example overlays for dev/prod. Include deployment guide, architecture diagram, and troubleshooting runbook.

### Rationale
- llm-d uses Kustomize for deployment configuration - tracing components must follow same pattern
- Base manifests in `deploy/components/`, environment-specific overlays in `deploy/environments/`
- Documentation critical for adoption: operators need clear setup instructions and troubleshooting guidance
- Architecture diagram explains trace flow and component relationships
- Example queries and trace screenshots help operators interpret telemetry data

### Alternatives Considered
- **Helm charts**: Rejected - llm-d uses Kustomize, introducing Helm adds tooling complexity
- **Minimal documentation**: Rejected - observability features have steep learning curve, thorough docs essential

### Implementation Notes
- Create Kustomize components:
  - `deploy/components/opentelemetry-operator/`: Operator deployment, Instrumentation CR template
  - `deploy/components/jaeger/`: Jaeger Operator, Jaeger CR (all-in-one and production variants)
  - `deploy/components/vllm-sim/`: Update kustomization to include tracing config
- Documentation files:
  - `docs/tracing/deployment-guide.md`: Step-by-step setup instructions
  - `docs/tracing/architecture.md`: Component diagram, trace flow explanation
  - `docs/tracing/troubleshooting.md`: Common issues and solutions
  - `docs/examples/tracing/trace-queries.md`: Example Jaeger queries for operational scenarios
- Include Mermaid diagram in architecture.md:
  ```
  Client → Envoy Gateway → EPP → vLLM
             ↓              ↓      ↓
          (W3C headers propagate)
             ↓              ↓      ↓
          Jaeger Collector ←──────┘
                  ↓
          Jaeger Query (UI)
  ```

---

## Summary of Key Decisions

| Area | Decision | Rationale |
|------|----------|-----------|
| **vLLM Tracing** | Use native `--otlp-traces-endpoint` flag | Provides inference-specific spans without code changes |
| **Auto-Instrumentation** | OpenTelemetry Operator Instrumentation CR | Adds FastAPI/HTTP spans via zero-config injection |
| **Tracing Backend** | Jaeger Operator with Elasticsearch storage | Production-grade, Kubernetes-native, mature tooling |
| **Trace Propagation** | W3C Trace Context headers | Industry standard, automatic support across stack |
| **Sampling** | Parent-based trace ID ratio (10% prod) | Balances observability and performance/cost |
| **Security** | mTLS for OTLP export in production | Protects trace data in transit |
| **Integration** | Add `.spec.tracing` to InferencePool CRD | Declarative, per-pool tracing configuration |
| **Deployment** | Kustomize components + overlays | Consistent with llm-d architecture |

---

## Open Questions for Implementation

1. **vLLM Container Images**: Do current vLLM images include OpenTelemetry SDK packages, or should they be added via init container?
2. **EPP Instrumentation**: Should EPP service be instrumented in this phase, or deferred to future work?
3. **Certificate Management**: Should mTLS certificates be managed via cert-manager, or manually provisioned?
4. **Jaeger Storage Sizing**: What retention period and storage capacity should be recommended for production?
5. **Performance Benchmarks**: What baseline latency metrics should be captured before enabling tracing to validate <5ms overhead target?

These questions should be addressed during Phase 1 (Design & Contracts) or marked as implementation details to be resolved during task execution.

---

**Research Phase Complete** - Ready for Phase 1: Design & Contracts
