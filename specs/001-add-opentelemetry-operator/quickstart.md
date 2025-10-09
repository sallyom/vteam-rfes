# Quickstart: OpenTelemetry Operator Instrumentation for llm-d Tracing

**Feature**: 001-add-opentelemetry-operator
**Date**: 2025-10-09
**Audience**: Operators, Platform Engineers, SREs

## Overview
This quickstart guide walks through deploying OpenTelemetry Operator and Jaeger to enable distributed tracing for llm-d inference workloads. By the end of this guide, you'll have end-to-end tracing from client requests through vLLM inference with traces visible in Jaeger UI.

**Prerequisites**:
- Kubernetes cluster 1.27+ with `kubectl` access
- llm-d inference scheduler installed (v0.5.0+)
- At least one InferencePool deployed
- Kustomize 5.0+

**Time to Complete**: 15-20 minutes

---

## Step 1: Deploy Observability Namespace

Create a dedicated namespace for observability components.

```bash
kubectl create namespace observability
```

**Verification**:
```bash
kubectl get namespace observability
# Expected: STATUS = Active
```

---

## Step 2: Deploy Jaeger Operator

Deploy the Jaeger Operator to manage Jaeger backend components.

```bash
cd deploy/components/jaeger

# Apply Jaeger Operator manifests
kubectl apply -f operator.yaml -n observability

# Wait for operator to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=jaeger-operator \
  -n observability --timeout=120s
```

**Verification**:
```bash
kubectl get deployment jaeger-operator -n observability
# Expected: READY = 1/1

kubectl logs -l app.kubernetes.io/name=jaeger-operator -n observability --tail=20
# Expected: No error messages, logs show "operator started"
```

---

## Step 3: Deploy Jaeger All-in-One Instance

Deploy Jaeger in all-in-one mode for development/testing (includes collector, query, and in-memory storage).

```bash
# Apply Jaeger instance CR
kubectl apply -f jaeger-instance-allinone.yaml -n observability

# Wait for Jaeger to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=jaeger \
  -n observability --timeout=120s
```

**Verification**:
```bash
kubectl get jaeger -n observability
# Expected: NAME = jaeger-llmd-tracing, STATUS = Running

kubectl get pods -n observability -l app.kubernetes.io/name=jaeger
# Expected: STATUS = Running

# Verify OTLP ports are exposed
kubectl get svc jaeger-llmd-tracing-collector -n observability
# Expected: Ports 4317 (otlp-grpc), 4318 (otlp-http)
```

**Access Jaeger UI**:
```bash
# Port-forward to Jaeger Query UI
kubectl port-forward -n observability svc/jaeger-llmd-tracing-query 16686:16686

# Open in browser: http://localhost:16686
# Expected: Jaeger UI loads, no services visible yet
```

---

## Step 4: Deploy OpenTelemetry Operator

Deploy the OpenTelemetry Operator to enable auto-instrumentation.

```bash
cd deploy/components/opentelemetry-operator

# Apply OpenTelemetry Operator CRDs and deployment
kubectl apply -f crds.yaml
kubectl apply -f operator.yaml

# Wait for operator to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=opentelemetry-operator \
  -n opentelemetry-operator-system --timeout=120s
```

**Verification**:
```bash
kubectl get deployment opentelemetry-operator -n opentelemetry-operator-system
# Expected: READY = 1/1

kubectl get crd instrumentations.opentelemetry.io
# Expected: Instrumentation CRD exists
```

---

## Step 5: Create Instrumentation Custom Resource

Deploy an Instrumentation CR to configure auto-instrumentation for vLLM pods.

```bash
# Switch to target namespace (where vLLM pods run)
export TARGET_NAMESPACE="llm-d-system"

# Apply Instrumentation CR
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: vllm-instrumentation
  namespace: $TARGET_NAMESPACE
spec:
  exporter:
    endpoint: "http://jaeger-llmd-tracing-collector.observability:4318"
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: "parentbased_traceidratio"
    argument: "1.0"  # 100% sampling for testing
  resource:
    addK8sUIDAttributes: true
  python:
    image: "ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.44b0"
    env:
      - name: OTEL_EXPORTER_OTLP_PROTOCOL
        value: "http/protobuf"
      - name: OTEL_TRACES_EXPORTER
        value: "otlp"
      - name: OTEL_METRICS_EXPORTER
        value: "none"
      - name: OTEL_LOGS_EXPORTER
        value: "none"
EOF
```

**Verification**:
```bash
kubectl get instrumentation vllm-instrumentation -n $TARGET_NAMESPACE
# Expected: Instrumentation CR exists
```

---

## Step 6: Enable Tracing for an InferencePool

Update an existing InferencePool to enable tracing, or create a new one.

```bash
# Option A: Update existing InferencePool
kubectl patch inferencepool my-gpt2-pool -n $TARGET_NAMESPACE --type=merge -p '
spec:
  tracing:
    enabled: true
    endpoint: "http://jaeger-llmd-tracing-collector.observability:4318"
    protocol: "http/protobuf"
    samplingRate: 1.0
    serviceName: "vllm-gpt2-pool"
'

# Option B: Create new InferencePool with tracing
kubectl apply -f - <<EOF
apiVersion: inference.llm-d.io/v1alpha1
kind: InferencePool
metadata:
  name: my-gpt2-pool
  namespace: $TARGET_NAMESPACE
spec:
  model: "gpt2"
  replicas: 1
  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      nvidia.com/gpu: "1"
  tracing:
    enabled: true
    endpoint: "http://jaeger-llmd-tracing-collector.observability:4318"
    protocol: "http/protobuf"
    samplingRate: 1.0
    serviceName: "vllm-gpt2-pool"
EOF
```

**Verification**:
```bash
# Wait for vLLM pods to restart and become ready
kubectl get pods -n $TARGET_NAMESPACE -l inference-pool=my-gpt2-pool
# Expected: STATUS = Running (may take 2-3 minutes for image pull and startup)

# Verify tracing configuration is applied
kubectl get pod -n $TARGET_NAMESPACE -l inference-pool=my-gpt2-pool -o yaml | grep -A 5 "otlp-traces-endpoint"
# Expected: --otlp-traces-endpoint=http://jaeger-llmd-tracing-collector.observability:4318

# Verify auto-instrumentation is injected
kubectl get pod -n $TARGET_NAMESPACE -l inference-pool=my-gpt2-pool -o yaml | grep -A 3 "opentelemetry-auto-instrumentation"
# Expected: Init container named "opentelemetry-auto-instrumentation" present
```

---

## Step 7: Generate Test Traces

Send test requests to vLLM to generate trace data.

```bash
# Port-forward to vLLM service
kubectl port-forward -n $TARGET_NAMESPACE svc/my-gpt2-pool 8000:8000 &

# Send a test completion request
curl -X POST http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt2",
    "prompt": "Once upon a time",
    "max_tokens": 50,
    "temperature": 0.7
  }'

# Expected: JSON response with completion text

# Send a few more requests to generate multiple traces
for i in {1..5}; do
  curl -X POST http://localhost:8000/v1/completions \
    -H "Content-Type: application/json" \
    -d "{\"model\": \"gpt2\", \"prompt\": \"Test prompt $i\", \"max_tokens\": 20}"
  sleep 1
done
```

**Verification**:
```bash
# Check vLLM logs for trace export confirmation
kubectl logs -n $TARGET_NAMESPACE -l inference-pool=my-gpt2-pool --tail=50 | grep -i "trace"
# Expected: Logs showing successful trace export (no errors)
```

---

## Step 8: View Traces in Jaeger UI

Open Jaeger UI to visualize traces.

```bash
# Ensure port-forward is running (from Step 3)
kubectl port-forward -n observability svc/jaeger-llmd-tracing-query 16686:16686

# Open in browser: http://localhost:16686
```

**In Jaeger UI**:
1. **Service dropdown**: Select `vllm-gpt2-pool`
2. **Find Traces button**: Click to load recent traces
3. **Expected**: List of traces, one per request sent in Step 7
4. **Click on a trace** to view details:
   - Should see 2-3 spans per trace:
     - `POST /v1/completions` (FastAPI auto-instrumentation)
     - `vllm.request` (vLLM native tracing)
     - Possibly `prefill` and `decode` spans
5. **Inspect span attributes**:
   - `http.method`, `http.status_code`
   - `llm.model=gpt2`
   - `llm.prompt.length`, `llm.completion.tokens`
   - `k8s.pod.name`, `k8s.namespace.name`

**Success Criteria**:
- ✅ Traces appear in Jaeger UI within 10 seconds of sending request
- ✅ Each trace contains multiple spans (FastAPI + vLLM)
- ✅ Spans have correct parent-child relationships (tree structure)
- ✅ Span attributes include HTTP, LLM, and Kubernetes metadata

---

## Step 9: Test Trace Context Propagation

Verify that trace context is propagated from client to vLLM.

```bash
# Send request with explicit traceparent header
curl -X POST http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -d '{
    "model": "gpt2",
    "prompt": "Trace propagation test",
    "max_tokens": 20
  }'

# In Jaeger UI, search for trace ID: 4bf92f3577b34da6a3ce929d0e0e4736
# Expected: Trace found with vLLM spans inheriting the provided trace ID
```

**Verification**:
- ✅ Trace ID in Jaeger matches the one sent in `traceparent` header
- ✅ vLLM spans are children of the client-provided parent span ID

---

## Step 10: Validate Performance Overhead

Measure latency with and without tracing to confirm <5ms overhead target.

```bash
# Benchmark with tracing enabled (current state)
for i in {1..100}; do
  curl -s -w "%{time_total}\n" -o /dev/null \
    -X POST http://localhost:8000/v1/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "gpt2", "prompt": "Test", "max_tokens": 10}'
done | awk '{sum+=$1; count++} END {print "Avg latency (with tracing): " sum/count " seconds"}'

# Disable tracing temporarily
kubectl patch inferencepool my-gpt2-pool -n $TARGET_NAMESPACE --type=merge -p '
spec:
  tracing:
    enabled: false
'

# Wait for pods to restart
kubectl wait --for=condition=ready pod -l inference-pool=my-gpt2-pool -n $TARGET_NAMESPACE --timeout=180s

# Benchmark without tracing
for i in {1..100}; do
  curl -s -w "%{time_total}\n" -o /dev/null \
    -X POST http://localhost:8000/v1/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "gpt2", "prompt": "Test", "max_tokens": 10}'
done | awk '{sum+=$1; count++} END {print "Avg latency (without tracing): " sum/count " seconds"}'

# Re-enable tracing
kubectl patch inferencepool my-gpt2-pool -n $TARGET_NAMESPACE --type=merge -p '
spec:
  tracing:
    enabled: true
'
```

**Success Criteria**:
- ✅ Latency difference is <5ms (0.005 seconds) on average
- ✅ p99 latency increase is <5ms (requires more sophisticated tooling like `hey` or `k6`)

---

## Troubleshooting

### Traces not appearing in Jaeger

**Symptoms**: No traces visible in Jaeger UI after sending requests.

**Checks**:
1. Verify Jaeger collector is running and healthy:
   ```bash
   kubectl logs -n observability -l app.kubernetes.io/component=collector --tail=50
   ```
2. Verify vLLM is exporting traces (check logs for OTLP export errors):
   ```bash
   kubectl logs -n $TARGET_NAMESPACE -l inference-pool=my-gpt2-pool --tail=100 | grep -i "otlp\|trace\|export"
   ```
3. Check network connectivity from vLLM pod to Jaeger collector:
   ```bash
   kubectl exec -n $TARGET_NAMESPACE -it <vllm-pod> -- curl -v http://jaeger-llmd-tracing-collector.observability:4318/v1/traces
   # Expected: Connection successful (even if 405 Method Not Allowed, connection is ok)
   ```

### Auto-instrumentation not injected

**Symptoms**: No `opentelemetry-auto-instrumentation` init container in vLLM pod.

**Checks**:
1. Verify Instrumentation CR exists in correct namespace:
   ```bash
   kubectl get instrumentation -n $TARGET_NAMESPACE
   ```
2. Verify OpenTelemetry Operator webhook is running:
   ```bash
   kubectl get mutatingwebhookconfiguration opentelemetry-operator-mutation
   ```
3. Check if pod has annotation:
   ```bash
   kubectl get pod -n $TARGET_NAMESPACE -l inference-pool=my-gpt2-pool -o jsonpath='{.items[0].metadata.annotations}'
   # Expected: instrumentation.opentelemetry.io/inject-python="true"
   ```

### High latency or performance degradation

**Symptoms**: Requests are significantly slower with tracing enabled.

**Solutions**:
1. Reduce sampling rate:
   ```bash
   kubectl patch inferencepool my-gpt2-pool -n $TARGET_NAMESPACE --type=merge -p '
   spec:
     tracing:
       samplingRate: 0.1  # Sample only 10% of traces
   '
   ```
2. Check span export queue depth and batch settings:
   ```bash
   kubectl set env deployment/my-gpt2-pool -n $TARGET_NAMESPACE \
     OTEL_BSP_MAX_EXPORT_BATCH_SIZE=512 \
     OTEL_BSP_SCHEDULE_DELAY=5000
   ```

---

## Next Steps

After completing this quickstart:

1. **Production Deployment**: Migrate from all-in-one Jaeger to production deployment with Elasticsearch backend (see `docs/tracing/production-deployment.md`)

2. **Multi-Pool Tracing**: Enable tracing for additional InferencePools with appropriate service names:
   ```bash
   kubectl patch inferencepool other-pool -n $TARGET_NAMESPACE --type=merge -p '
   spec:
     tracing:
       enabled: true
       endpoint: "http://jaeger-llmd-tracing-collector.observability:4318"
       serviceName: "vllm-other-pool"
   '
   ```

3. **Advanced Queries**: Learn to use Jaeger's query capabilities to diagnose issues (see `docs/examples/tracing/trace-queries.md`)

4. **Security Hardening**: Enable mTLS for OTLP export in production (see `docs/tracing/security.md`)

5. **Integration**: Connect Jaeger with Grafana for unified observability dashboards

---

## Validation Checklist

Before considering this feature complete, verify:

- [ ] Jaeger Operator deployed and running
- [ ] Jaeger all-in-one instance running, UI accessible
- [ ] OpenTelemetry Operator deployed and running
- [ ] Instrumentation CR created in target namespace
- [ ] InferencePool updated with tracing configuration
- [ ] vLLM pods restarted with tracing args and auto-instrumentation
- [ ] Test requests sent to vLLM
- [ ] Traces visible in Jaeger UI with correct service name
- [ ] Traces contain multiple spans (FastAPI + vLLM native)
- [ ] Span attributes include HTTP, LLM, and Kubernetes metadata
- [ ] Trace context propagation works (tested with manual traceparent header)
- [ ] Performance overhead <5ms (benchmarked)

**Quickstart Complete!** 🎉

You now have distributed tracing enabled for llm-d inference workloads. Refer to the full documentation in `docs/tracing/` for advanced configuration, troubleshooting, and production best practices.
