# Data Model: OpenTelemetry Operator Instrumentation for llm-d Tracing

**Feature**: 001-add-opentelemetry-operator
**Date**: 2025-10-09

## Overview
This document defines the key entities, data structures, and relationships for the OpenTelemetry tracing feature. Since this is an infrastructure feature primarily involving Kubernetes custom resources and configuration, the "data model" focuses on CRD specifications, configuration schemas, and trace data structures.

---

## 1. InferencePool CRD Extension

**Purpose**: Add optional tracing configuration to the existing InferencePool CRD to enable per-pool tracing control.

### Schema Addition
```yaml
apiVersion: inference.llm-d.io/v1alpha1
kind: InferencePool
metadata:
  name: example-pool
spec:
  # ... existing fields ...

  # NEW: Tracing configuration (optional)
  tracing:
    # Enable/disable tracing for this pool
    enabled: boolean

    # OTLP endpoint for trace export
    # Examples:
    #   - "grpc://jaeger-collector.observability:4317"
    #   - "http://jaeger-collector.observability:4318"
    endpoint: string

    # Transport protocol: "grpc" or "http/protobuf"
    # Default: "grpc"
    protocol: string

    # Sampling rate (0.0 to 1.0)
    # 0.0 = no sampling, 1.0 = sample all traces
    # Default: 1.0 (dev), 0.1 (prod)
    samplingRate: float

    # Service name for traces
    # Default: "vllm-{pool-name}"
    serviceName: string

    # Additional resource attributes to attach to spans
    # Example: {"environment": "production", "tenant": "acme"}
    resourceAttributes: map[string]string

    # Enable insecure connections (no TLS)
    # Use only in dev/test environments
    # Default: false
    insecure: boolean

    # TLS certificate configuration (optional)
    tls:
      # CA certificate for verifying collector
      caCertSecret: string  # Secret name containing ca.crt

      # Client certificate for mTLS (optional)
      clientCertSecret: string  # Secret name containing tls.crt, tls.key
```

### Validation Rules
- `enabled`: Required field when `tracing` block is present
- `endpoint`: Required when `enabled: true`, must be valid URL with `grpc://` or `http://` scheme
- `protocol`: Must be `"grpc"` or `"http/protobuf"`, defaults to `"grpc"`
- `samplingRate`: Must be between 0.0 and 1.0 (inclusive)
- `serviceName`: Must match pattern `[a-z0-9]([-a-z0-9]*[a-z0-9])?` (DNS-1123 label)
- `insecure`: Cannot be `true` when `tls` block is present
- `tls.caCertSecret` / `tls.clientCertSecret`: Must reference existing Secret in same namespace

### Default Values
```yaml
tracing:
  enabled: false  # Tracing disabled by default
  protocol: "grpc"
  samplingRate: 1.0  # Can be overridden by environment-specific patches
  serviceName: "vllm-{{ .metadata.name }}"
  insecure: false
```

### State Transitions
- **Disabled → Enabled**: Controller creates vLLM deployment with tracing args/env vars, pods restart
- **Enabled → Disabled**: Controller removes tracing configuration from deployment, pods restart
- **Endpoint Changed**: Pods restart with new endpoint configuration
- **Sampling Rate Changed**: Pods restart with new `OTEL_TRACES_SAMPLER_ARG` value

---

## 2. Instrumentation Custom Resource

**Purpose**: OpenTelemetry Operator Instrumentation CR that configures auto-instrumentation for vLLM pods.

### Schema
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: vllm-instrumentation
  namespace: llm-d-system
spec:
  # Trace exporter configuration
  exporter:
    endpoint: "http://jaeger-collector.observability:4318"

  # Trace context propagators
  propagators:
    - tracecontext  # W3C Trace Context
    - baggage       # W3C Baggage

  # Sampling configuration
  sampler:
    type: "parentbased_traceidratio"
    argument: "0.1"  # 10% sampling

  # Resource attributes attached to all spans
  resource:
    addK8sUIDAttributes: true  # Add pod/namespace metadata
    resourceAttributes:
      deployment.environment: "production"
      service.namespace: "llm-d"

  # Python auto-instrumentation configuration
  python:
    # Auto-instrumentation image
    image: "ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.44b0"

    # Environment variables for instrumentation
    env:
      - name: OTEL_EXPORTER_OTLP_PROTOCOL
        value: "http/protobuf"
      - name: OTEL_TRACES_EXPORTER
        value: "otlp"
      - name: OTEL_METRICS_EXPORTER
        value: "none"  # Disable metrics for now
      - name: OTEL_LOGS_EXPORTER
        value: "none"  # Disable logs for now
```

### Key Fields
- `exporter.endpoint`: OTLP endpoint for trace export (Jaeger collector)
- `propagators`: List of context propagators (W3C Trace Context required for Envoy compatibility)
- `sampler.type`: Sampling algorithm (`always_on`, `always_off`, `traceidratio`, `parentbased_traceidratio`)
- `sampler.argument`: Sampling configuration (for `traceidratio`: value between 0.0-1.0)
- `python.image`: Container image with OpenTelemetry Python auto-instrumentation libraries
- `resource.addK8sUIDAttributes`: Automatically add Kubernetes pod/namespace metadata to spans

### Pod Annotation
vLLM pods must be annotated to trigger instrumentation injection:
```yaml
metadata:
  annotations:
    instrumentation.opentelemetry.io/inject-python: "true"
```

---

## 3. Jaeger Custom Resource

**Purpose**: Defines Jaeger backend deployment (collector, query, storage) managed by Jaeger Operator.

### Schema - Development Configuration
```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-llmd-tracing
  namespace: observability
spec:
  strategy: allInOne  # Single pod with collector, query, and in-memory storage

  ingress:
    enabled: true
    hosts:
      - jaeger-llmd.example.com

  storage:
    type: memory
    options:
      memory:
        max-traces: 10000  # Store up to 10k traces in memory

  collector:
    maxReplicas: 1
    resources:
      limits:
        cpu: "1"
        memory: "1Gi"

  query:
    resources:
      limits:
        cpu: "500m"
        memory: "512Mi"
```

### Schema - Production Configuration
```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-llmd-tracing
  namespace: observability
spec:
  strategy: production  # Separate collector, query, and persistent storage

  ingress:
    enabled: true
    hosts:
      - jaeger-llmd.example.com
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod

  storage:
    type: elasticsearch
    options:
      es:
        server-urls: "http://elasticsearch:9200"
        index-prefix: "jaeger-llmd"
        num-shards: 3
        num-replicas: 1
    secretName: jaeger-es-credentials  # Contains ES username/password

  collector:
    maxReplicas: 5
    autoscale: true
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "2Gi"

  query:
    replicas: 2
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "1"
        memory: "1Gi"

  # Retention policy (requires ES curator or ILM)
  # Implemented via separate CronJob or ES ILM policy
```

### Key Fields
- `strategy`: Deployment mode (`allInOne` for dev, `production` for prod)
- `storage.type`: Backend storage (`memory`, `elasticsearch`, `cassandra`, `kafka`)
- `storage.options`: Storage-specific configuration (connection URLs, indices, credentials)
- `collector.maxReplicas`: Horizontal scaling for trace ingestion
- `query.replicas`: Number of query service instances for high availability

---

## 4. Trace Span Schema

**Purpose**: Defines the structure of OpenTelemetry trace spans exported by vLLM and auto-instrumentation.

### vLLM Native Span Example
```json
{
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "parentSpanId": "00f067aa0ba902b6",
  "name": "vllm.request",
  "kind": "SERVER",
  "startTime": 1696838400000000000,
  "endTime": 1696838401234000000,
  "duration": 1234000000,
  "status": {
    "code": "OK"
  },
  "attributes": {
    "service.name": "vllm-gpt2-pool",
    "http.method": "POST",
    "http.url": "/v1/completions",
    "http.status_code": 200,
    "llm.model": "gpt2",
    "llm.request.type": "completions",
    "llm.prompt.length": 512,
    "llm.completion.tokens": 128,
    "llm.prefill.duration_ms": 450,
    "llm.decode.duration_ms": 784,
    "llm.kv_cache.hit_rate": 0.85,
    "k8s.pod.name": "vllm-gpt2-pool-78d9f-xvz7k",
    "k8s.namespace.name": "llm-d-system"
  }
}
```

### FastAPI Auto-Instrumentation Span Example
```json
{
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b6",
  "parentSpanId": "00f067aa0ba902b5",
  "name": "POST /v1/completions",
  "kind": "SERVER",
  "startTime": 1696838400000000000,
  "endTime": 1696838401234000000,
  "duration": 1234000000,
  "status": {
    "code": "OK"
  },
  "attributes": {
    "service.name": "vllm-gpt2-pool",
    "http.method": "POST",
    "http.route": "/v1/completions",
    "http.status_code": 200,
    "http.request.header.content_type": "application/json",
    "http.response.header.content_type": "application/json",
    "net.peer.ip": "10.244.0.15",
    "net.peer.port": 8000
  }
}
```

### Span Attributes Taxonomy
**Standard HTTP Attributes** (from OpenTelemetry semantic conventions):
- `http.method`: HTTP method (GET, POST, etc.)
- `http.url`: Full request URL
- `http.status_code`: HTTP response status
- `net.peer.ip`: Client IP address

**LLM-Specific Attributes** (from OpenTelemetry AI/ML semantic conventions):
- `llm.model`: Model name/ID
- `llm.request.type`: Request type (completions, chat, embeddings)
- `llm.prompt.length`: Input prompt length in tokens
- `llm.completion.tokens`: Generated tokens count
- `llm.prefill.duration_ms`: Prefill phase duration
- `llm.decode.duration_ms`: Decode phase duration
- `llm.kv_cache.hit_rate`: KV cache hit rate (0.0-1.0)

**Kubernetes Attributes** (from auto-instrumentation):
- `k8s.pod.name`: Pod name
- `k8s.namespace.name`: Namespace
- `k8s.deployment.name`: Deployment name
- `k8s.node.name`: Node name

---

## 5. Configuration Schemas

### OTLP Endpoint Configuration
**Format**: `{protocol}://{host}:{port}[{path}]`

**Examples**:
- gRPC: `grpc://jaeger-collector.observability:4317`
- HTTP: `http://jaeger-collector.observability:4318`
- HTTP with path: `http://jaeger-collector.observability:4318/v1/traces`

**Environment Variables** (generated by controller):
```bash
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT="grpc://jaeger-collector.observability:4317"
OTEL_EXPORTER_OTLP_TRACES_PROTOCOL="grpc"  # or "http/protobuf"
OTEL_EXPORTER_OTLP_TRACES_INSECURE="false"
OTEL_SERVICE_NAME="vllm-gpt2-pool"
OTEL_TRACES_SAMPLER="parentbased_traceidratio"
OTEL_TRACES_SAMPLER_ARG="0.1"
```

### Sampling Configuration Schema
```yaml
sampler:
  # Sampler type
  type: "parentbased_traceidratio"

  # Sampler-specific argument
  # - always_on: no argument
  # - always_off: no argument
  # - traceidratio: "0.1" (sample 10% of traces)
  # - parentbased_traceidratio: "0.1" (inherit parent decision, else 10%)
  argument: "0.1"
```

**Supported Sampler Types**:
1. `always_on`: Sample all traces (100%)
2. `always_off`: Sample no traces (0%)
3. `traceidratio`: Probabilistic sampling (e.g., "0.1" = 10%)
4. `parentbased_traceidratio`: Parent-based with fallback to ratio (recommended)
5. `parentbased_always_on`: Parent-based with fallback to always-on
6. `parentbased_always_off`: Parent-based with fallback to always-off

---

## 6. Entity Relationships

```
InferencePool (CRD)
  └─ spec.tracing.enabled = true
      ├─→ vLLM Deployment (generated)
      │    ├─ --otlp-traces-endpoint arg
      │    ├─ OTEL_* environment variables
      │    └─ annotation: instrumentation.opentelemetry.io/inject-python: "true"
      │
      ├─→ Instrumentation CR (namespace-scoped)
      │    ├─ Targets vLLM pods via label selectors
      │    └─ Injects auto-instrumentation init container
      │
      └─→ Jaeger Collector (receives traces)
           └─→ Jaeger Storage Backend (Elasticsearch)
                └─→ Jaeger Query UI (visualization)

Trace Flow:
  Client Request
    └─ Envoy Gateway (propagates traceparent header)
         └─ vLLM Pod
              ├─ FastAPI Handler (auto-instrumented span)
              │   └─ Exports to Jaeger via OTLP
              │
              └─ vLLM Inference (native tracing span)
                  └─ Exports to Jaeger via OTLP
```

---

## 7. Data Lifecycle

### Trace Data Flow
1. **Generation**: vLLM processes request, generates spans (FastAPI + native)
2. **Export**: Spans batched and sent to Jaeger collector via OTLP (async)
3. **Ingestion**: Jaeger collector receives spans, validates, writes to storage
4. **Storage**: Spans persisted in Elasticsearch with index prefix `jaeger-llmd-{date}`
5. **Retention**: Elasticsearch ILM policy deletes indices older than 30 days (configurable)
6. **Query**: Operators search traces via Jaeger Query UI by service, operation, tags, time range

### Configuration Lifecycle
1. **Creation**: Operator creates InferencePool with `spec.tracing.enabled: true`
2. **Reconciliation**: Controller generates vLLM deployment with tracing configuration
3. **Injection**: OpenTelemetry Operator mutates pod spec to add instrumentation
4. **Runtime**: vLLM starts, reads OTEL_* env vars, begins exporting traces
5. **Update**: Operator modifies tracing config (e.g., sampling rate) → pods restart
6. **Deletion**: Operator deletes InferencePool → tracing stops, existing traces retained per retention policy

---

## Summary

This data model defines:
- **InferencePool CRD extension** for declarative tracing configuration
- **Instrumentation CR** for auto-injection of OpenTelemetry SDK
- **Jaeger CR** for tracing backend deployment
- **Trace span schema** with LLM-specific attributes
- **Configuration schemas** for OTLP endpoints and sampling
- **Entity relationships** showing data flow from pods to UI

All schemas follow Kubernetes API conventions and OpenTelemetry semantic conventions for interoperability.
