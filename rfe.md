# OpenTelemetry Operator Instrumentation for llm-d Tracing

**Feature Overview:**

Integrate OpenTelemetry (OTel) Operator auto-instrumentation with vLLM's native tracing capabilities to provide comprehensive, distributed observability across the llm-d inference scheduler stack. This feature will deploy an OpenTelemetry Instrumentation custom resource (CR) alongside a Jaeger deployment to enable end-to-end tracing from inference requests through Envoy gateway, EPP routing decisions, and vLLM model serving workloads. This provides llm-d operators with unified visibility into request latency, routing behavior, and model inference performance, enabling faster troubleshooting, performance optimization, and capacity planning.

**Goals:**

* **Unified Observability**: Provide a single, cohesive tracing solution that captures the entire inference request lifecycle—from client request through Envoy gateway, EPP routing decisions, to vLLM model serving and response generation.

* **Operator-Driven Deployment**: Leverage the OpenTelemetry Operator's Instrumentation CR to automatically instrument target workloads without requiring manual code changes or application restarts, simplifying deployment and maintenance.

* **Deep vLLM Integration**: Combine OpenTelemetry Operator auto-instrumentation with vLLM's native `--otlp-traces-endpoint` tracing to capture both framework-level (FastAPI, HTTP handlers) and model-specific (prefill/decode, KV cache operations) telemetry.

* **Production-Ready Observability Backend**: Deploy Jaeger as the tracing backend to provide visualization, trace aggregation, and query capabilities for llm-d operators and SREs.

* **Minimal Performance Overhead**: Ensure that tracing instrumentation adds negligible latency (<5ms p99) to inference requests and scales to handle production workloads.

**Target users**: Platform engineers, SREs, and ML infrastructure teams operating llm-d inference clusters who need to diagnose routing issues, identify performance bottlenecks, and optimize resource utilization.

**Current state vs. future state**: Today, llm-d operators have limited visibility into why specific routing decisions are made, where latency is introduced in the request path, or how efficiently vLLM pods are serving requests. With this feature, operators gain comprehensive distributed traces that connect client requests through the entire inference stack, enabling data-driven optimization and rapid incident resolution.

**Out of Scope:**

* Metrics and logging collection (this RFE focuses exclusively on distributed tracing)
* Custom trace exporters beyond OTLP (only gRPC and HTTP/protobuf OTLP protocols are supported)
* Application Performance Monitoring (APM) features such as error tracking, service dependency graphs, or alerting (these may be addressed in future RFEs)
* Auto-instrumentation for languages other than Python (vLLM and related workloads are Python-based)
* Long-term trace storage and retention policies (operators are expected to configure Jaeger storage backends separately)
* Integration with non-Jaeger tracing backends (e.g., Tempo, Zipkin) in the initial implementation

**Requirements:**

* **[MVP]** Deploy OpenTelemetry Operator with Instrumentation CR that targets vLLM pods and automatically injects OTel auto-instrumentation for Python/FastAPI
* **[MVP]** Deploy Jaeger all-in-one or production-ready Jaeger deployment (operator, collector, query, storage) in the llm-d cluster
* **[MVP]** Configure vLLM pods to use `--otlp-traces-endpoint` flag to export native vLLM tracing spans to the Jaeger collector
* **[MVP]** Enable trace context propagation from client requests through Envoy gateway to vLLM pods using W3C Trace Context headers
* **[MVP]** Capture spans for key operations: EPP routing decisions, Envoy ext-proc calls, vLLM request handling, prefill/decode phases, and KV cache operations
* **[MVP]** Provide example manifests and documentation for deploying the OTel Operator, Instrumentation CR, and Jaeger components
* **[MVP]** Validate that traces correctly link spans across service boundaries (client → Envoy → EPP → vLLM)
* Configure OTel Collector pipeline for trace sampling, batching, and export to Jaeger with recommended production settings
* Support both gRPC and HTTP/protobuf OTLP export protocols
* Provide ConfigMap-based configuration for OTel instrumentation settings (sampling rate, exporter endpoint, resource attributes)
* Document performance impact and recommended resource limits for instrumented workloads
* Integrate with existing llm-d deployment tooling (Kustomize-based component structure)

**Done - Acceptance Criteria:**

* An operator can deploy the OpenTelemetry Operator and Instrumentation CR using the provided manifests and see auto-instrumentation applied to vLLM pods
* vLLM pods automatically export traces to Jaeger without requiring custom code changes or container image modifications
* Traces are visible in the Jaeger UI showing the complete request path from ingress through EPP routing to vLLM inference
* Each trace includes spans for: HTTP request handling (auto-instrumented), EPP routing decision (if instrumented), vLLM request processing (native tracing), and prefill/decode operations
* Trace context is correctly propagated across service boundaries using W3C Trace Context headers
* Documentation is provided covering: deployment instructions, configuration options, troubleshooting common issues, and interpreting traces
* Performance testing demonstrates <5ms p99 latency overhead introduced by tracing instrumentation under typical inference workloads
* Example queries and trace patterns are documented for common operational scenarios (e.g., "find slow inference requests", "identify routing anomalies", "analyze KV cache hit rates")

**Use Cases - i.e. User Experience & Workflow:**

**Use Case 1: Deploy OpenTelemetry Instrumentation**

*Main Success Scenario:*
1. Operator deploys OpenTelemetry Operator to the cluster using provided manifest
2. Operator creates Instrumentation CR targeting vLLM pods (using label selectors)
3. OpenTelemetry Operator injects auto-instrumentation init containers and environment variables into matching pods
4. vLLM pods restart and automatically begin exporting traces to the configured OTLP endpoint
5. Operator verifies successful instrumentation by checking pod logs and Jaeger UI

*Alternative Flow 1: Instrumentation fails due to incompatible Python version*
- Operator checks compatibility matrix and updates vLLM container image to supported Python version

*Alternative Flow 2: Instrumentation CR doesn't match any pods*
- Operator reviews label selectors in Instrumentation CR and corrects to match vLLM pod labels

**Use Case 2: Deploy and Configure Jaeger Backend**

*Main Success Scenario:*
1. Operator deploys Jaeger using provided manifests (Jaeger Operator or all-in-one)
2. Operator configures Jaeger collector to accept OTLP traces on port 4317 (gRPC) and 4318 (HTTP)
3. Operator configures Instrumentation CR to point to Jaeger collector OTLP endpoint
4. Operator accesses Jaeger UI via Service/Ingress and verifies services are appearing

*Alternative Flow 1: Production deployment requires persistent storage*
- Operator configures Jaeger with Elasticsearch or Cassandra storage backend
- Operator validates trace retention policies

**Use Case 3: Investigate Slow Inference Request**

*Main Success Scenario:*
1. User reports slow inference response (>5s latency)
2. Operator opens Jaeger UI and searches for traces from the affected service within the time window
3. Operator identifies the trace corresponding to the slow request
4. Operator drills into trace spans to identify bottleneck (e.g., long prefill time, slow KV cache lookup, queueing in EPP)
5. Operator uses span metadata (model name, request parameters, pod ID) to diagnose root cause
6. Operator takes corrective action (e.g., scale up pods, adjust routing weights, optimize model configuration)

**Use Case 4: Analyze EPP Routing Decisions**

*Main Success Scenario:*
1. Operator wants to understand why certain requests are routed to specific vLLM pods
2. Operator searches Jaeger for traces containing EPP spans
3. Operator examines EPP span attributes showing scoring decisions, filter results, and selected pod
4. Operator correlates routing decisions with downstream vLLM performance metrics
5. Operator adjusts EPP plugin configuration based on observed routing patterns

**Use Case 5: Monitor vLLM Prefill/Decode Performance**

*Main Success Scenario:*
1. Operator wants to understand the distribution of prefill vs. decode latency
2. Operator queries Jaeger for traces containing vLLM spans with operation names like "prefill" and "decode"
3. Operator analyzes span durations and tags (prompt length, output tokens, KV cache hit/miss)
4. Operator identifies opportunities for optimization (e.g., KV cache tuning, P/D disaggregation)

**Documentation Considerations:**

* Deployment guide covering OpenTelemetry Operator installation, Instrumentation CR configuration, and Jaeger deployment
* Configuration reference documenting all supported environment variables, sampling strategies, and exporter options
* Architecture diagram showing trace flow from client through Envoy, EPP, and vLLM with span relationships
* Troubleshooting guide for common issues: instrumentation not injecting, traces not appearing, context propagation failures, high overhead
* Performance tuning guide covering sampling rates, batch sizes, and resource limits
* Example trace scenarios with annotated screenshots showing how to interpret EPP routing decisions and vLLM performance metrics
* Integration guide for connecting Jaeger with existing observability tools (Grafana, Prometheus)
* Security considerations for OTLP endpoints, Jaeger UI access, and sensitive data in traces
* Migration guide if extending existing vLLM OpenTelemetry POC (see `vllm/examples/online_serving/opentelemetry/`)

**Questions to answer:**

* Should the Instrumentation CR target only vLLM pods, or should we also auto-instrument EPP and Envoy?
* What sampling strategy should be used by default? (always-on for development, probabilistic/rate-limited for production?)
* Should we deploy Jaeger all-in-one for simplicity, or Jaeger Operator for production-grade deployment?
* How should trace context be propagated through Envoy ext-proc? Does Envoy automatically forward trace headers to EPP?
* What storage backend should be recommended for Jaeger in production? (in-memory, Elasticsearch, Cassandra, S3)
* How should we handle trace data that may contain sensitive information (prompts, completions)?
* Should we provide a separate OTel Collector deployment for advanced pipeline configuration, or rely on direct export to Jaeger?
* What resource limits should be set for auto-instrumentation overhead? (CPU, memory impact on vLLM pods)
* How do we handle version compatibility between OpenTelemetry SDK, vLLM native tracing, and FastAPI auto-instrumentation?
* Should the Instrumentation CR be deployed per-namespace, per-InferencePool, or cluster-wide?
* How do we correlate traces with existing Prometheus metrics from vLLM and EPP?
* What trace attributes should be added to spans to aid in debugging (model name, InferencePool, pod ID, etc.)?
* Should we provide pre-built dashboards or saved queries for common operational scenarios?

**Background & Strategic Fit:**

OpenTelemetry has emerged as the industry standard for cloud-native observability, with broad adoption across Kubernetes ecosystems and vendor-neutral tooling. vLLM has recently added native OpenTelemetry tracing support (see `vllm/examples/online_serving/opentelemetry/`), providing detailed instrumentation of model serving operations including prefill/decode phases, KV cache operations, and request queueing.

The llm-d inference scheduler is a complex distributed system involving multiple components (Envoy gateway, EPP routing service, vLLM pods) where request latency and routing behavior can be difficult to diagnose without comprehensive tracing. By combining OpenTelemetry Operator auto-instrumentation with vLLM's native tracing, we can provide operators with deep visibility into the entire inference stack without requiring manual instrumentation or code changes.

This feature aligns with llm-d's goal of providing production-ready, enterprise-grade inference infrastructure. Distributed tracing is a critical capability for operating ML serving systems at scale, enabling data-driven optimization, rapid incident response, and capacity planning.

The OpenTelemetry Operator's Instrumentation CR approach provides a Kubernetes-native, declarative way to manage tracing instrumentation, consistent with llm-d's architecture philosophy of leveraging upstream Kubernetes projects (Gateway API, GIE) and avoiding custom controllers where possible.

**Customer Considerations:**

* **Multi-tenancy**: Customers running multi-tenant llm-d clusters may need to isolate trace data by namespace or tenant. Consider supporting per-namespace Instrumentation CRs and trace filtering/routing in the OTel Collector.

* **Compliance and Privacy**: Traces may contain sensitive data (user prompts, model outputs). Customers in regulated industries may require trace data to be scrubbed, encrypted at rest, or retained for limited periods. Document recommendations for trace data governance.

* **Performance Sensitivity**: Inference workloads are latency-sensitive. Customers may have strict SLAs (e.g., p99 < 100ms). Provide clear guidance on sampling strategies, expected overhead, and how to tune instrumentation for minimal impact.

* **Hybrid/On-Prem Deployments**: Customers running llm-d in air-gapped or on-premises environments may not have access to managed tracing services. Ensure that Jaeger can be deployed fully within the cluster without external dependencies.

* **Existing Observability Stacks**: Customers may already have investments in other tracing backends (Datadog, New Relic, Honeycomb) or APM tools. While this RFE focuses on Jaeger, document how to configure OTLP export to alternative backends.

* **Cost Considerations**: Trace data volume can grow rapidly in high-throughput environments. Provide guidance on cost-effective sampling, storage, and retention strategies. Consider supporting tail-based sampling for capturing only "interesting" traces (errors, slow requests).

* **Skill Sets**: Not all customers have expertise in OpenTelemetry or distributed tracing. Provide opinionated defaults, example configurations, and runbooks to lower the barrier to adoption.

* **Integration with Existing Workflows**: Customers may want to link traces to incident management systems (PagerDuty, Opsgenie), dashboards (Grafana), or log aggregation (Loki). Document integration patterns and provide examples.
