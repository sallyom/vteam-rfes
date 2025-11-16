# OpenTelemetry Tracing Enhancement Recommendations for llm-d Components

**Date:** 2025-11-16
**Components Analyzed:** gateway-api-inference-extension, vllm/vllm/v1
**Purpose:** Add comprehensive OpenTelemetry tracing to improve observability across llm-d inference stack

## Executive Summary

This document provides recommendations for adding custom OpenTelemetry (OTel) spans to the gateway-api-inference-extension and vllm/vllm/v1 components of the llm-d distributed inference system. Both components have basic OTel SDK initialization but minimal custom instrumentation. Enhanced tracing will provide end-to-end observability across the inference request lifecycle, from gateway routing through model execution.

## Architecture Overview

### llm-d System Architecture

The llm-d system is a Kubernetes-native distributed inference stack with the following key components:

1. **Inference Gateway (IGW)** - Built on Envoy proxy with gateway-api-inference-extension
2. **Inference Scheduler** - LLM-optimized load balancing with prefix-cache awareness
3. **vLLM Model Servers** - Distributed inference engines (can be disaggregated into prefill/decode roles)
4. **KV Cache Manager** - Manages distributed KV cache
5. **Routing Sidecar** - Coordinates disaggregated serving

**Request Flow:**
```
Client Request → Envoy Proxy → Gateway Extension (EPP) → Scheduler → vLLM Pod(s) → Response
                      ↓                                        ↓
                Flow Control                            Model Execution
                Queue Management                        Token Generation
                Load Balancing                          KV Cache Operations
```

### Current OpenTelemetry Implementation

#### gateway-api-inference-extension (Go)

**Location:** `pkg/common/telemetry.go:46-140`

**Current Implementation:**
- OTel SDK initialized in `InitTracing()` function
- Supports OTLP gRPC exporter and console/stdout exporter
- Configurable via environment variables:
  - `OTEL_SERVICE_NAME` (default: "gateway-api-inference-extension")
  - `OTEL_EXPORTER_OTLP_ENDPOINT` (default: "http://localhost:4317")
  - `OTEL_TRACES_EXPORTER` (default: "console")
  - `OTEL_TRACES_SAMPLER` (default: "parentbased_traceidratio")
  - `OTEL_TRACES_SAMPLER_ARG` (default: "0.1")
- Uses W3C TraceContext and Baggage propagation
- Enabled via `--tracing` flag in `cmd/epp/runner/runner.go:130`

**Existing Spans:**
- Currently NO custom spans found in the codebase
- Only SDK initialization without span creation

#### vllm/vllm/v1 (Python)

**Location:** `vllm/v1/engine/async_llm.py:134-138`, `vllm/v1/engine/llm_engine.py:115-119`

**Current Implementation:**
- OTel tracer initialized via `vllm.tracing.init_tracer()` when `observability_config.otlp_traces_endpoint` is set
- Tracer attached to `OutputProcessor` instance
- Ray metrics integration with OTel-compatible metric names (`vllm/v1/metrics/ray_wrappers.py`)

**Existing Spans:**
- **Location:** `vllm/v1/engine/output_processor.py:488-550`
- Single span: `"llm_request"` with SpanKind.SERVER
- Created when request finishes
- **Attributes tracked:**
  - Latency metrics: TTFT, E2E, queue time, prefill time, decode time, inference time
  - Usage metrics: prompt tokens, completion tokens
  - Request parameters: request_id, top_p, max_tokens, temperature, n
- Uses trace context extraction from request headers
- Timestamp at arrival time (nanoseconds)

## Gap Analysis

### Missing Observability in Gateway Extension

The gateway extension handles critical routing, scheduling, and flow control decisions but has **zero custom spans**. Key gaps:

1. **No request lifecycle tracking** - Cannot observe request flow through EPP
2. **No scheduling decision visibility** - Cannot see why specific pods were selected
3. **No flow control observability** - Cannot track queuing, admission control, or saturation
4. **No pod selection tracing** - Cannot debug routing decisions
5. **No error path tracking** - Cannot trace failures through the system

### Missing Observability in vLLM v1

The vLLM v1 engine has only a single span at request completion. Key gaps:

1. **No engine core operation tracking** - Cannot observe scheduling, batching, execution phases
2. **No token generation tracking** - Cannot measure per-iteration performance
3. **No KV cache operation visibility** - Cannot track cache hits, misses, transfers
4. **No prefill/decode phase separation** - Cannot distinguish these critical phases
5. **No executor/worker communication tracking** - Cannot observe distributed execution
6. **No batch formation tracking** - Cannot see how requests are grouped

## Recommendations

### Gateway Extension Span Strategy

#### 1. Request Processing Pipeline Spans

**Parent Span: `gateway.request`**
- **Location:** `pkg/epp/handlers/server.go:132` (Process method)
- **Duration:** Full request lifecycle through gateway
- **Attributes:**
  - `request.id` - Generated or from headers
  - `request.model` - Incoming model name
  - `request.size_bytes` - Request payload size
  - `fairness.id` - Fairness group identifier
  - `objective.key` - Inference objective key
  - `pool.name` - Target InferencePool name
  - `pool.namespace` - InferencePool namespace

**Child Spans:**

1. **`gateway.request.headers`**
   - **Location:** `pkg/epp/handlers/server.go` - Header processing
   - **Attributes:**
     - `header.count`
     - `metadata.extracted` (boolean)

2. **`gateway.request.body`**
   - **Location:** `pkg/epp/handlers/server.go` - Body processing
   - **Attributes:**
     - `body.chunks` - Number of body chunks
     - `body.total_bytes`
     - `body.parsed` (boolean)

3. **`gateway.director.handle_request`**
   - **Location:** `pkg/epp/requestcontrol/director.go`
   - **Attributes:**
     - `admission.result` (admitted/rejected/queued)
     - `queue.size` - Current queue depth
     - `saturation.detected` (boolean)

#### 2. Scheduling and Routing Spans

**Span: `gateway.scheduler.schedule`**
- **Location:** `pkg/epp/scheduling/scheduler.go`
- **Attributes:**
  - `scheduler.policy` - Scheduling policy used
  - `candidates.count` - Number of candidate pods
  - `filter.applied` - Filters applied
  - `scorer.applied` - Scorers applied
  - `selected.pod` - Pod name selected
  - `selected.score` - Final score

**Child Spans:**

1. **`gateway.scheduler.filter`**
   - **Location:** Scheduling framework filter plugins
   - **Attributes:**
     - `filter.name`
     - `filter.rejected_count`
     - `filter.duration_ms`

2. **`gateway.scheduler.score`**
   - **Location:** Scheduling framework scorer plugins
   - **Attributes:**
     - `scorer.name`
     - `scorer.type` (kvcache/queue/lora_affinity)
     - `score.values` - JSON array of pod scores
     - `scorer.duration_ms`

3. **`gateway.scheduler.pick`**
   - **Location:** Picker plugin (max_score/random/weighted_random)
   - **Attributes:**
     - `picker.type`
     - `picker.selected_index`

#### 3. Flow Control Spans (Experimental Feature)

**Span: `gateway.flowcontrol.enqueue`**
- **Location:** `pkg/epp/flowcontrol/controller/controller.go`
- **Attributes:**
  - `flow.key` - Flow identifier
  - `flow.priority` - Priority level
  - `queue.position` - Position in queue
  - `ttl.effective` - Effective TTL
  - `byte_size` - Request byte size

**Span: `gateway.flowcontrol.dispatch`**
- **Location:** Flow control dispatch policies
- **Attributes:**
  - `dispatch.policy` (fcfs/roundrobin/besthead)
  - `wait.duration_ms` - Time spent waiting
  - `dispatch.result` (dispatched/timeout/cancelled)

#### 4. Pod Metrics Collection Spans

**Span: `gateway.metrics.collect`**
- **Location:** `pkg/epp/backend/metrics/pod_metrics.go`
- **Attributes:**
  - `pod.name`
  - `pod.ip`
  - `metrics.scheme` (http/https)
  - `metrics.scrape_duration_ms`
  - `metrics.success` (boolean)
  - `metrics.kv_cache_utilization`
  - `metrics.queue_depth`

#### 5. Response Processing Spans

**Span: `gateway.response.process`**
- **Location:** `pkg/epp/handlers/server.go` - Response handlers
- **Attributes:**
  - `response.status_code`
  - `response.streaming` (boolean)
  - `response.chunks` - Number of chunks
  - `response.total_bytes`
  - `response.complete` (boolean)

### vLLM v1 Span Strategy

#### 1. Engine Core Request Processing

**Parent Span: `vllm.engine.request`** (Enhance existing `llm_request`)
- **Location:** `vllm/v1/engine/async_llm.py:340` (_add_request method)
- **Duration:** Request admission to completion
- **Additional Attributes:**
  - `request.type` (generate/encode/pooling)
  - `request.priority`
  - `request.data_parallel_rank`
  - `request.has_parent` (boolean for n>1)
  - `lora.name` - LoRA adapter name if used

**Child Spans:**

1. **`vllm.processor.process_inputs`**
   - **Location:** `vllm/v1/engine/processor.py`
   - **Attributes:**
     - `tokenization.prompt_length`
     - `tokenization.truncated` (boolean)
     - `multimodal.present` (boolean)
     - `processor.duration_ms`

2. **`vllm.engine.add_to_core`**
   - **Location:** `vllm/v1/engine/async_llm.py:352`
   - **Attributes:**
     - `core.queue_size` - Current request queue size

#### 2. Engine Core Scheduling Spans

**Span: `vllm.scheduler.schedule`**
- **Location:** `vllm/v1/core/sched/scheduler.py` (schedule method)
- **Attributes:**
  - `scheduler.type` - Scheduler class name
  - `scheduler.phase` (prefill/decode/mixed)
  - `batch.prefill_requests`
  - `batch.decode_requests`
  - `batch.total_tokens`
  - `chunked_prefill.enabled` (boolean)
  - `kv_cache.blocks_allocated`
  - `kv_cache.blocks_available`

**Child Spans:**

1. **`vllm.scheduler.admit_requests`**
   - **Location:** Scheduler admission logic
   - **Attributes:**
     - `requests.waiting`
     - `requests.admitted`
     - `requests.rejected`
     - `blocks.needed`
     - `blocks.available`

2. **`vllm.scheduler.allocate_blocks`**
   - **Location:** KV cache block allocation
   - **Attributes:**
     - `allocation.blocks_allocated`
     - `allocation.blocks_freed`
     - `allocation.cache_hit` (boolean)
     - `prefix_cache.blocks_reused`

3. **`vllm.scheduler.create_batch`**
   - **Location:** Batch formation logic
   - **Attributes:**
     - `batch.size`
     - `batch.total_tokens`
     - `batch.max_seq_len`

#### 3. Model Execution Spans

**Span: `vllm.executor.execute_model`**
- **Location:** `vllm/v1/executor/` (execute_model methods)
- **Attributes:**
  - `executor.type` (uniproc/multiproc/ray)
  - `executor.workers` - Number of workers
  - `batch.prefill_size`
  - `batch.decode_size`
  - `model.forward_duration_ms`

**Child Spans:**

1. **`vllm.worker.prepare_inputs`**
   - **Location:** Worker input preparation
   - **Attributes:**
     - `input_metadata.created`
     - `attention.backend` (flash/xformers/etc)

2. **`vllm.model.forward`**
   - **Location:** Model forward pass
   - **Attributes:**
     - `model.layers` - Number of layers
     - `model.hidden_size`
     - `compute.duration_ms`

3. **`vllm.sampler.sample`**
   - **Location:** Token sampling
   - **Attributes:**
     - `sampling.temperature`
     - `sampling.top_p`
     - `sampling.tokens_sampled`
     - `sampling.duration_ms`

#### 4. KV Cache Transfer Spans (Disaggregated Serving)

**Span: `vllm.kv_cache.transfer`**
- **Location:** KV cache connector operations
- **Attributes:**
  - `transfer.direction` (send/receive)
  - `transfer.protocol` (rdma/tcp/nixl)
  - `transfer.blocks`
  - `transfer.bytes`
  - `transfer.duration_ms`
  - `transfer.source_pod`
  - `transfer.dest_pod`

#### 5. Output Processing Spans

**Span: `vllm.output.process_batch`**
- **Location:** `vllm/v1/engine/output_processor.py:383` (process_outputs method)
- **Attributes:**
  - `batch.outputs_processed`
  - `detokenization.duration_ms`
  - `logprobs.computed` (boolean)

**Child Spans:**

1. **`vllm.detokenizer.update`**
   - **Location:** Detokenizer update per request
   - **Attributes:**
     - `tokens.new` - New tokens to detokenize
     - `stop_string.detected` (boolean)

2. **`vllm.logprobs.compute`**
   - **Location:** Logprobs processor
   - **Attributes:**
     - `logprobs.prompt` (boolean)
     - `logprobs.sample` (boolean)

## Implementation Guidelines

### Best Practices

1. **Span Naming Convention**
   - Gateway: `gateway.<component>.<operation>`
   - vLLM: `vllm.<component>.<operation>`
   - Use lowercase with dots as separators

2. **Attribute Naming Convention**
   - Use semantic conventions where applicable (OpenTelemetry spec)
   - Component-specific: `<component>.<metric_name>`
   - Existing vLLM attributes use custom naming (keep consistent)

3. **Sampling Strategy**
   - Gateway: Parent-based sampling with configurable ratio (default 0.1)
   - vLLM: Inherit sampling from gateway via trace context propagation
   - High-volume operations (per-token) should be sampled more aggressively

4. **Context Propagation**
   - Gateway extracts W3C TraceContext from incoming HTTP headers
   - Gateway must inject trace context into vLLM requests (HTTP headers)
   - vLLM extracts trace context from headers (already implemented)
   - Ensure baggage propagation for cross-service metadata

5. **Performance Considerations**
   - Use `otel.WithSpanKind()` appropriately (SERVER, CLIENT, INTERNAL)
   - Record exceptions in spans: `span.RecordError(err)`
   - Set span status: `span.SetStatus(codes.Error, "description")`
   - Avoid high-cardinality attributes (e.g., request_id only in span name or event)
   - Use events for point-in-time occurrences within a span

6. **Error Handling**
   - Always record errors in spans before returning
   - Include error type, message, and stack trace as events
   - Set span status to ERROR
   - Don't let tracing errors break primary request flow

### Code Patterns

#### Gateway (Go)

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/trace"
)

// In pkg/epp/handlers/server.go
func (s *StreamingServer) Process(srv extProcPb.ExternalProcessor_ProcessServer) error {
    ctx := srv.Context()
    tracer := otel.Tracer("gateway-api-inference-extension")

    ctx, span := tracer.Start(ctx, "gateway.request",
        trace.WithSpanKind(trace.SpanKindServer),
    )
    defer span.End()

    // Set initial attributes
    span.SetAttributes(
        attribute.String("request.id", reqID),
        attribute.String("pool.name", poolName),
    )

    // Process request...

    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }

    span.SetStatus(codes.Ok, "")
    return nil
}

// For child spans
func (d *Director) HandleRequest(ctx context.Context, reqCtx *RequestContext) error {
    tracer := otel.Tracer("gateway-api-inference-extension")
    ctx, span := tracer.Start(ctx, "gateway.director.handle_request")
    defer span.End()

    // Logic...
}
```

#### vLLM (Python)

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode

# In vllm/v1/engine/core.py or scheduler
def schedule(self) -> SchedulerOutput:
    if self.tracer:
        with self.tracer.start_as_current_span(
            "vllm.scheduler.schedule",
            kind=trace.SpanKind.INTERNAL
        ) as span:
            span.set_attribute("scheduler.type", self.__class__.__name__)

            try:
                output = self._schedule_internal()
                span.set_attribute("batch.total_tokens", output.total_tokens)
                span.set_status(Status(StatusCode.OK))
                return output
            except Exception as e:
                span.record_exception(e)
                span.set_status(Status(StatusCode.ERROR, str(e)))
                raise
    else:
        return self._schedule_internal()
```

### Integration Points

1. **Gateway → vLLM Trace Propagation**
   - Gateway must inject trace context into HTTP request headers to vLLM
   - Headers: `traceparent`, `tracestate` (W3C TraceContext)
   - Location: `pkg/epp/handlers/server.go` - Before proxying to backend pod
   - Use `otel.GetTextMapPropagator().Inject(ctx, carrier)`

2. **vLLM Trace Context Extraction**
   - Already implemented: `vllm/v1/engine/output_processor.py:499`
   - Uses `vllm.tracing.extract_trace_context(engine_core_output.trace_headers)`
   - Ensure trace headers are passed through entire request pipeline

3. **Cross-Component Correlation**
   - Use trace_id to correlate gateway and vLLM spans
   - Gateway should log trace_id for debugging
   - vLLM should include trace_id in error logs

## Phased Implementation Plan

### Phase 1: Core Request Lifecycle (High Priority)

**Gateway:**
- `gateway.request` - Top-level request span
- `gateway.director.handle_request` - Admission control
- `gateway.scheduler.schedule` - Pod selection
- `gateway.response.process` - Response handling

**vLLM:**
- Enhance existing `llm_request` span with additional attributes
- `vllm.scheduler.schedule` - Batch scheduling
- `vllm.executor.execute_model` - Model execution

**Deliverables:**
- End-to-end trace from gateway to vLLM
- Basic latency breakdown (queue, schedule, execute)
- Error tracking across components

### Phase 2: Detailed Scheduling and Execution (Medium Priority)

**Gateway:**
- `gateway.scheduler.filter` - Filter plugins
- `gateway.scheduler.score` - Scorer plugins
- `gateway.metrics.collect` - Pod metrics collection

**vLLM:**
- `vllm.scheduler.admit_requests` - Request admission
- `vllm.scheduler.allocate_blocks` - KV cache allocation
- `vllm.output.process_batch` - Output processing
- `vllm.detokenizer.update` - Detokenization

**Deliverables:**
- Scheduler decision visibility
- KV cache operation tracking
- Token generation performance metrics

### Phase 3: Advanced Features (Lower Priority)

**Gateway:**
- `gateway.flowcontrol.enqueue` - Flow control queuing
- `gateway.flowcontrol.dispatch` - Flow control dispatch

**vLLM:**
- `vllm.kv_cache.transfer` - Disaggregated KV cache transfers
- `vllm.worker.prepare_inputs` - Worker input prep
- `vllm.model.forward` - Model forward pass (may need sampling)

**Deliverables:**
- Flow control observability
- Disaggregated serving traces
- Fine-grained performance profiling

## Metrics vs. Traces

**Use Traces for:**
- Request-scoped operations (single request flow)
- Latency breakdown and bottleneck identification
- Error propagation and debugging
- Cross-service correlation

**Use Metrics for:**
- Aggregated statistics (throughput, error rates)
- Resource utilization (KV cache, GPU memory)
- SLO monitoring (P50, P95, P99 latencies)
- Alerting thresholds

**Use Logs for:**
- Detailed error messages and stack traces
- Debugging information
- Audit trails
- Events without request context

## Langfuse Integration Considerations

If adding Langfuse tracing in addition to OpenTelemetry:

1. **Langfuse Spans** should capture:
   - LLM call metadata (model, params)
   - Prompt and completion content
   - Cost tracking (tokens × price)
   - User/session information

2. **Langfuse vs. OTel Separation:**
   - OTel: Technical performance and infrastructure observability
   - Langfuse: LLM-specific application observability (prompts, quality)

3. **Dual Instrumentation:**
   - Can run both simultaneously
   - Correlate via shared request_id
   - OTel for ops team, Langfuse for ML/app team

4. **Implementation:**
   - vLLM layer is better suited for Langfuse (has prompt/completion)
   - Gateway layer should focus on OTel (routing/infrastructure)

## Testing and Validation

### Unit Testing
- Mock tracer for unit tests
- Verify span creation and attributes
- Test error recording

### Integration Testing
- Run with Jaeger or Zipkin backend
- Verify end-to-end traces
- Check trace context propagation
- Validate attribute completeness

### Performance Testing
- Measure overhead with tracing enabled/disabled
- Target: <5% latency overhead at 10% sampling
- Monitor memory usage

### Production Rollout
1. Deploy with low sampling rate (1%)
2. Monitor for performance impact
3. Gradually increase sampling
4. Tune based on trace volume and storage costs

## References

- OpenTelemetry Go: https://opentelemetry.io/docs/languages/go/
- OpenTelemetry Python: https://opentelemetry.io/docs/languages/python/
- Semantic Conventions: https://opentelemetry.io/docs/specs/semconv/
- W3C TraceContext: https://www.w3.org/TR/trace-context/
- llm-d Architecture: https://github.com/llm-d/llm-d
- vLLM Documentation: https://docs.vllm.ai
- Gateway API Inference Extension: https://gateway-api-inference-extension.sigs.k8s.io/

## Appendix A: Key File Locations

### Gateway Extension
- OTel initialization: `pkg/common/telemetry.go:46-140`
- EPP server: `pkg/epp/handlers/server.go`
- Director: `pkg/epp/requestcontrol/director.go`
- Scheduler: `pkg/epp/scheduling/scheduler.go`
- Flow control: `pkg/epp/flowcontrol/controller/controller.go`

### vLLM v1
- OTel initialization: `vllm/v1/engine/async_llm.py:134-138`, `vllm/v1/engine/llm_engine.py:115-119`
- Existing span: `vllm/v1/engine/output_processor.py:488-550`
- Engine core: `vllm/v1/engine/core.py`
- Scheduler: `vllm/v1/core/sched/scheduler.py`
- Executor: `vllm/v1/executor/`
- Output processor: `vllm/v1/engine/output_processor.py`

## Appendix B: Environment Variables

### Gateway Extension
```bash
OTEL_SERVICE_NAME=gateway-api-inference-extension
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_TRACES_EXPORTER=otlp  # or "console"
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1  # 10% sampling
```

### vLLM
```python
# Set in VllmConfig.observability_config
observability_config.otlp_traces_endpoint = "http://localhost:4317"
```

## Appendix C: Example Jaeger Query

To view a complete request trace:
```
service="gateway-api-inference-extension" OR service="vllm.llm_engine"
```

Filter by high latency:
```
duration > 1s
```

Find errors:
```
error=true
```
