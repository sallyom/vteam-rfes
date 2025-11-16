# Distributed Tracing for llm-d

## Summary

This proposal introduces distributed tracing for llm-d distributed inference framework. Distributed tracing will provide observability into inference
workloads, enabling performance optimization, cost control, and quality validation across the llm-d stack. The solution will be built on OpenTelemetry
with **manual trace initialization and custom spans** to provide rich, end-to-end insights across all components.

## Motivation

LLM inference workloads present unique observability challenges due to their expensive, non-uniform, and often slow request patterns. In distributed
systems like llm-d, understanding request flow across components like the inference scheduler, KV cache manager, and vLLM instances becomes critical
for operationalizing inference at scale.

Current monitoring approaches lack the granular, request-level visibility needed to optimize Time to First Token (TTFT), Inter-Token Latency (ITL),
and cost efficiency in complex serving topologies involving disaggregated serving, KV-cache aware routing, and multi-model deployments.

### Goals

* **Enhanced Performance Diagnostics**: Provide detailed, request-level visibility into llm-d bottlenecks, enabling optimization of TTFT,
ITL, and overall throughput across distributed serving components through targeted custom spans.

* **Cost Efficiency and Attribution**: Enable per-request token usage tracking and cost attribution across applications and workloads, crucial for
managing high LLM computational costs.

* **Quality and Accuracy Validation**: Enable validation of response quality and performance characteristics across complex RAG pipelines, while
maintaining strict data privacy by avoiding sensitive payload exposure.

* **Simplified Debugging**: Provide end-to-end request tracing across llm-d components with detailed custom spans at critical decision points
to reduce mean time to resolution (MTTR) for performance degradation and error scenarios.

* **Optimization Validation**: Provide concrete, per-request data to validate the effectiveness of llm-d's advanced optimizations like KV-cache aware
routing, disaggregated serving, and intelligent scheduling through comprehensive instrumentation.

* **Component-Specific Insights**: Enable component owners to instrument critical code paths with custom spans that expose internal operations,
decision-making processes, and performance characteristics specific to their domain.

### Non-Goals

* **Metrics Collection**: This proposal focuses on distributed tracing, not metrics collection, though OpenTelemetry collectors can export to both.
Note that OpenTelemetry instrumentation can emit metrics data from instrumented processes, for example, with HTTP servers. Tracing gives users
important RED metrics without direct metrics instrumentation.

* **Log Aggregation**: While OpenTelemetry supports logs, this proposal addresses distributed tracing only.

* **Real-time Alerting**: Tracing data analysis and alerting are out of scope, although the metrics emitted from trace data can feed into alerting systems.

* **SLO and SLA Guarantees**: Initial implementation focuses on observability rather than SLA enforcement, though tracing data
supports SLO and SLA validation.

* **Sensitive Data Exposure**: This proposal does not include request/response payload tracing to prevent inadvertent logging of sensitive LLM inputs/outputs.
Token counts and metadata are captured without exposing actual content.

## Proposal

This proposal introduces distributed tracing as a unified opt-in capability across the llm-d stack,
implemented through **manual OpenTelemetry instrumentation** with explicit trace initialization and custom span creation. The solution focuses on instrumenting
the critical path of LLM inference requests with targeted spans at key decision points to provide end-to-end observability from inference gateway to model response.

Unlike auto-instrumentation approaches, this manual instrumentation strategy allows precise control over what operations are traced, what attributes
are captured, and where performance overhead is acceptable. Each component will initialize its own tracer and create custom spans around operations
that provide meaningful observability value.

### User Stories

#### Story 1

As a platform operator running llm-d in production, I want to quickly identify which component in my distributed serving
pipeline is causing high latency, including which specific operation (scheduling, KV cache lookup, model execution) so that I can
properly identify root-cause of the problem, optimize resource allocation, and meet my SLAs.

#### Story 2

As a cost-conscious organization using llm-d, I want to track token usage and costs per application and request type, along with
KV cache hit rates and reuse statistics, so that I can optimize my prompt engineering and model selection to reduce operational expenses.

#### Story 3

As an llm-d developer validating new optimizations, I want to measure the impact of KV-cache aware routing and P/D disaggregation on request
latency with detailed timing for each phase (admission, scheduling, prefill, decode, cache operations) so that I can quantify the benefits
of these advanced features.

#### Story 4

As an llm-d developer/administrator, I have noticed a significant change in performance since the last upgrade. I want to compare the execution,
caching and decision-making in routing between the two releases by examining detailed scheduler spans, filter/scorer timings, and cache operation metrics.

#### Story 5

As a developer debugging a failed inference request, I want to see the complete trace showing which scheduler filters rejected which pods,
what scores were assigned, which pod was selected, and at what point the failure occurred, so I can quickly identify the root cause.

## Design Details

### llm-d Stack

The tracing solution will be based on **OpenTelemetry**, an open,
vendor-agnostic standard for collecting and generating telemetry data. OpenTelemetry offers:

- Semantic conventions for GenAI operations
- Standardized attributes for LLM-related telemetry
- Broad ecosystem support and vendor neutrality
- Mature SDKs for Go and Python with manual instrumentation support

#### Resources

* [OpenTelemetry traces documentation](https://opentelemetry.io/docs/concepts/signals/traces/)
* [OpenTelemetry semantic conventions for GenAI](https://github.com/open-telemetry/semantic-conventions/blob/main/model/gen-ai/spans.yaml)
* [GenAI semantic conventions for GenAI systems documentation](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
* [OpenTelemetry Go Manual Instrumentation](https://opentelemetry.io/docs/languages/go/instrumentation/)
* [OpenTelemetry Python Manual Instrumentation](https://opentelemetry.io/docs/languages/python/instrumentation/)

### Component Instrumentation Strategy

The instrumentation strategy focuses on the critical path of LLM inference requests through llm-d,
covering key components responsible for routing, caching, and serving with **custom spans at strategic points**.

**Implementation Approach:**
Each component will implement **manual trace initialization** using the OpenTelemetry SDK and create **custom spans** around:
- Request entry and exit points (end-to-end visibility)
- Critical decision points (scheduling, routing, admission control)
- Expensive operations (cache lookups, model execution, KV transfers)
- Error paths and exceptional conditions

**Manual Instrumentation Benefits:**
- **Precise Control**: Component owners decide exactly what to trace and when
- **Rich Context**: Custom attributes can expose component-specific decision-making details
- **Performance Awareness**: Instrumentation can be added only where overhead is acceptable
- **Security by Design**: Explicit control over what data enters spans prevents accidental exposure
- **Debugging Power**: Detailed spans at key operations enable rapid root cause analysis

### Current Implementation Status

#### gateway-api-inference-extension (Go)
- **Status**: OTel SDK initialized, **zero custom spans**
- **Location**: `pkg/common/telemetry.go:46-140`
- **Initialization**: `InitTracing()` sets up OTLP exporter, W3C propagation, parent-based sampling
- **Configuration**: Environment variables (`OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, etc.)
- **Gap**: No spans created despite having full SDK infrastructure

#### vLLM v1 (Python)
- **Status**: Tracer initialized, **one custom span** at request completion
- **Location**: Tracer init in `vllm/v1/engine/async_llm.py:134-138`, span in `vllm/v1/engine/output_processor.py:488-550`
- **Initialization**: `init_tracer("vllm.llm_engine", observability_config.otlp_traces_endpoint)`
- **Existing span**: `llm_request` (SERVER) with TTFT, queue time, prefill/decode time, token counts, request params
- **Gap**: Only end-to-end span, no visibility into scheduler, executor, or per-iteration operations

### Proposed Custom Spans by Component

## Components

### **`gateway-api-inference-extension (Inference Gateway)`**

**Component Architecture**: The gateway-api-inference-extension implements the Endpoint Picker Protocol (EPP), operating as a gRPC service that
receives routing requests from Envoy proxy and makes intelligent endpoint selection decisions based on sophisticated scheduling policies.

**Current Tracing State**:
- ✅ OTel SDK fully initialized (`pkg/common/telemetry.go`)
- ❌ **Zero custom spans implemented**
- Supports OTLP gRPC and console exporters
- Parent-based sampling with configurable ratio (default 10%)

**Proposed Manual Instrumentation**:

#### Parent Span: `gateway.request`
- **Location**: `pkg/epp/handlers/server.go:132` (Process method)
- **Kind**: `trace.SpanKindServer`
- **Duration**: Full request lifecycle through gateway
- **Attributes**:
  - `request.id` - Generated or extracted request ID
  - `request.model` - Incoming model name
  - `request.size_bytes` - Request payload size
  - `fairness.id` - Fairness group identifier
  - `objective.key` - Inference objective key
  - `pool.name` - Target InferencePool name
  - `pool.namespace` - InferencePool namespace
  - `request.streaming` - Whether response is streaming
- **Error Handling**: Record exceptions, set span status to ERROR on failures
- **Context**: Extract from incoming gRPC metadata, propagate to children

#### Child Spans:

**1. `gateway.request.headers`**
- **Location**: `pkg/epp/handlers/server.go` - Header processing phase
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `header.count` - Number of headers processed
  - `metadata.extracted` - Whether required metadata was found
- **Purpose**: Track header parsing overhead and metadata extraction success

**2. `gateway.request.body`**
- **Location**: `pkg/epp/handlers/server.go` - Body processing phase
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `body.chunks` - Number of body chunks received
  - `body.total_bytes` - Total request body size
  - `body.parsed` - Whether body parsing succeeded
- **Purpose**: Measure body processing time and identify large request overhead

**3. `gateway.director.handle_request`**
- **Location**: `pkg/epp/requestcontrol/director.go`
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `admission.result` - admitted/rejected/queued
  - `queue.size` - Current queue depth at admission
  - `saturation.detected` - Whether system is saturated
  - `flow_control.enabled` - Whether experimental flow control is active
- **Purpose**: Track admission control decisions and queue state

**4. `gateway.scheduler.schedule`**
- **Location**: `pkg/epp/scheduling/scheduler.go`
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `scheduler.policy` - Scheduling policy used
  - `candidates.initial` - Initial number of candidate pods
  - `candidates.after_filter` - Candidates remaining after filters
  - `selected.pod_name` - Pod name selected
  - `selected.pod_ip` - Pod IP selected
  - `selected.score` - Final score of selected pod
  - `scheduler.duration_ms` - Time spent in scheduling
- **Purpose**: Core scheduling decision visibility - critical for debugging routing
- **Child Spans**:

  **4a. `gateway.scheduler.filter`**
  - **Location**: Scheduling framework filter plugins
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `filter.name` - Filter plugin name
    - `filter.type` - Filter type (e.g., header-based-testing)
    - `filter.rejected_count` - Number of pods rejected
    - `filter.duration_ms` - Filter execution time
  - **Purpose**: Identify which filters are rejecting pods and their performance impact
  - **Note**: One span per filter execution

  **4b. `gateway.scheduler.score`**
  - **Location**: Scheduling framework scorer plugins
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `scorer.name` - Scorer plugin name
    - `scorer.type` - kvcache_utilization/queue/lora_affinity
    - `scorer.weights` - Scorer weights (JSON)
    - `score.min` - Minimum score assigned
    - `score.max` - Maximum score assigned
    - `score.avg` - Average score assigned
    - `scorer.duration_ms` - Scorer execution time
  - **Purpose**: Understand scoring distribution and identify slow scorers
  - **Note**: One span per scorer execution

  **4c. `gateway.scheduler.pick`**
  - **Location**: Picker plugin (max_score/random/weighted_random)
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `picker.type` - max_score/random/weighted_random
    - `picker.selected_index` - Index of selected pod in scored list
  - **Purpose**: Track final selection logic

**5. `gateway.flowcontrol.enqueue`** (when experimental flow control enabled)
- **Location**: `pkg/epp/flowcontrol/controller/controller.go`
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `flow.key` - Flow identifier
  - `flow.priority` - Priority level
  - `queue.position` - Position in queue
  - `ttl.effective_ms` - Effective TTL in milliseconds
  - `request.byte_size` - Request byte size
- **Purpose**: Track queuing behavior in flow control system

**6. `gateway.flowcontrol.dispatch`** (when experimental flow control enabled)
- **Location**: Flow control dispatch policies
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `dispatch.policy` - fcfs/roundrobin/besthead
  - `wait.duration_ms` - Time spent waiting in queue
  - `dispatch.result` - dispatched/timeout/cancelled
- **Purpose**: Measure queue wait times and dispatch outcomes

**7. `gateway.metrics.collect`**
- **Location**: `pkg/epp/backend/metrics/pod_metrics.go`
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `pod.name` - Pod name
  - `pod.ip` - Pod IP
  - `metrics.scheme` - http/https
  - `metrics.scrape_duration_ms` - Time to scrape metrics
  - `metrics.success` - Whether scrape succeeded
  - `metrics.kv_cache_utilization` - Current KV cache utilization
  - `metrics.queue_depth` - Current queue depth
- **Purpose**: Track metrics collection overhead and freshness
- **Note**: May want to sample this span more aggressively due to frequency

**8. `gateway.response.process`**
- **Location**: `pkg/epp/handlers/server.go` - Response handlers
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `response.status_code` - HTTP status code
  - `response.streaming` - Whether streaming response
  - `response.chunks` - Number of chunks received
  - `response.total_bytes` - Total response size
  - `response.complete` - Whether response completed successfully
- **Purpose**: Track response processing and identify large response overhead

**9. `gateway.backend.proxy`**
- **Location**: When proxying to backend vLLM pod
- **Kind**: `trace.SpanKindClient`
- **Attributes**:
  - `backend.pod_name` - Target pod name
  - `backend.pod_ip` - Target pod IP
  - `backend.port` - Target port
  - `http.method` - HTTP method
  - `http.url` - Request URL (without query params)
- **Purpose**: Measure network latency to backend pods
- **Context**: **Must inject trace context into HTTP headers to vLLM**

**Implementation Priority**:
1. High: `gateway.request`, `gateway.scheduler.schedule`, `gateway.director.handle_request`, `gateway.backend.proxy`
2. Medium: Scheduler child spans (filter, score, pick), `gateway.response.process`
3. Low: Flow control spans, `gateway.metrics.collect`

---

### **`vLLM v1 Instances`**

**Current Tracing State**:
- ✅ Tracer initialized via `init_tracer()` when `otlp_traces_endpoint` configured
- ✅ One custom span: `llm_request` at request completion with comprehensive attributes
- ❌ No visibility into scheduler, executor, or iteration-level operations

**Component Architecture**: vLLM v1 is the inference engine that handles model execution, scheduling, batching, and token generation.
Key subsystems include:
- **AsyncLLM/LLMEngine**: Request orchestration
- **Processor**: Input tokenization and processing
- **Scheduler**: Batch scheduling and KV cache allocation
- **Executor**: Model execution coordination across workers
- **OutputProcessor**: Detokenization and result assembly

**Proposed Manual Instrumentation**:

#### Enhanced Parent Span: `vllm.engine.request`
- **Current**: `llm_request` span in `output_processor.py:488-550`
- **Enhancement**: Move to `async_llm.py:340` (_add_request method) to capture full lifecycle
- **Kind**: `trace.SpanKindServer`
- **Duration**: Request admission to completion
- **Existing Attributes** (keep):
  - `gen_ai.request.id` - Request ID
  - `gen_ai.usage.prompt_tokens` - Prompt token count
  - `gen_ai.usage.completion_tokens` - Completion token count
  - `gen_ai.latency.time_to_first_token` - TTFT
  - `gen_ai.latency.e2e` - End-to-end time
  - `gen_ai.latency.time_in_queue` - Queue time
  - `gen_ai.latency.time_in_model.prefill` - Prefill time
  - `gen_ai.latency.time_in_model.decode` - Decode time
  - `gen_ai.request.top_p`, `gen_ai.request.max_tokens`, `gen_ai.request.temperature`, `gen_ai.request.n`
- **New Attributes**:
  - `gen_ai.request.type` - generate/encode/pooling
  - `gen_ai.request.priority` - Request priority
  - `vllm.request.data_parallel_rank` - DP rank if specified
  - `vllm.request.has_parent` - Boolean for n>1 parallel sampling
  - `vllm.lora.name` - LoRA adapter name if used
  - `vllm.kv_cache.num_cached_tokens` - Number of cached tokens reused
- **Context**: Extract from `trace_headers`, propagate to children

#### New Child Spans:

**1. `vllm.processor.process_inputs`**
- **Location**: `vllm/v1/engine/processor.py` - Processor.process_inputs()
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `vllm.tokenization.prompt_tokens` - Tokenized prompt length
  - `vllm.tokenization.truncated` - Whether prompt was truncated
  - `vllm.multimodal.present` - Whether multimodal data present
  - `vllm.processor.duration_ms` - Processing time
- **Purpose**: Track tokenization overhead and identify truncation

**2. `vllm.engine.add_to_core`**
- **Location**: `vllm/v1/engine/async_llm.py:352` - After processor, before scheduler
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `vllm.core.queue_size` - Current engine core queue size
  - `vllm.core.requests_running` - Number of requests currently running
- **Purpose**: Measure queueing into engine core

**3. `vllm.scheduler.schedule`**
- **Location**: `vllm/v1/core/sched/scheduler.py` - Main schedule method
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `vllm.scheduler.type` - Scheduler class name
  - `vllm.scheduler.iteration` - Iteration number
  - `vllm.batch.phase` - prefill/decode/mixed
  - `vllm.batch.prefill_requests` - Number of prefill requests in batch
  - `vllm.batch.decode_requests` - Number of decode requests in batch
  - `vllm.batch.total_tokens` - Total tokens in batch
  - `vllm.batch.max_seq_len` - Maximum sequence length in batch
  - `vllm.chunked_prefill.enabled` - Whether chunked prefill used
  - `vllm.kv_cache.blocks_allocated` - Blocks allocated this iteration
  - `vllm.kv_cache.blocks_available` - Blocks available before allocation
  - `vllm.kv_cache.blocks_freed` - Blocks freed this iteration
  - `vllm.scheduler.duration_ms` - Scheduling time
- **Purpose**: Core scheduling visibility - critical for understanding batching and cache behavior
- **Child Spans**:

  **3a. `vllm.scheduler.admit_requests`**
  - **Location**: Scheduler admission logic
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `vllm.requests.waiting` - Requests in waiting queue
    - `vllm.requests.admitted` - Requests admitted this iteration
    - `vllm.requests.preempted` - Requests preempted
    - `vllm.blocks.needed` - Blocks needed for waiting requests
    - `vllm.blocks.available` - Blocks available for allocation
  - **Purpose**: Track admission decisions and preemption

  **3b. `vllm.scheduler.allocate_blocks`**
  - **Location**: KV cache block allocation
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `vllm.allocation.blocks_allocated` - Blocks allocated
    - `vllm.allocation.blocks_freed` - Blocks freed
    - `vllm.prefix_cache.hit` - Whether prefix cache hit occurred
    - `vllm.prefix_cache.blocks_reused` - Number of blocks reused from cache
    - `vllm.allocation.duration_ms` - Allocation time
  - **Purpose**: Measure prefix caching effectiveness

  **3c. `vllm.scheduler.create_batch`**
  - **Location**: Batch formation logic
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `vllm.batch.size` - Number of requests in batch
    - `vllm.batch.total_tokens` - Total tokens
    - `vllm.batch.max_seq_len` - Max sequence length
    - `vllm.batch.num_prefill_tokens` - Tokens in prefill phase
    - `vllm.batch.num_decode_tokens` - Tokens in decode phase
  - **Purpose**: Understand batch composition

**4. `vllm.executor.execute_model`**
- **Location**: `vllm/v1/executor/*` - execute_model methods
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `vllm.executor.type` - uniproc/multiproc/ray/ray_distributed
  - `vllm.executor.num_workers` - Number of workers
  - `vllm.batch.prefill_size` - Prefill batch size
  - `vllm.batch.decode_size` - Decode batch size
  - `vllm.model.forward_duration_ms` - Model forward pass time
  - `vllm.executor.duration_ms` - Total executor time
- **Purpose**: Measure model execution overhead and identify slow forwards
- **Child Spans**:

  **4a. `vllm.worker.prepare_inputs`**
  - **Location**: Worker input preparation
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `vllm.input_metadata.created` - Whether input metadata created
    - `vllm.attention.backend` - flash_attn/xformers/naive/etc
  - **Purpose**: Track input prep overhead

  **4b. `vllm.model.forward`**
  - **Location**: Model forward pass
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `vllm.model.name` - Model name
    - `vllm.model.num_layers` - Number of layers
    - `vllm.model.hidden_size` - Hidden size
    - `vllm.compute.duration_ms` - Compute time
  - **Purpose**: Measure model compute time
  - **Note**: May want to sample aggressively due to frequency

  **4c. `vllm.sampler.sample`**
  - **Location**: Token sampling
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `vllm.sampling.temperature` - Temperature used
    - `vllm.sampling.top_p` - Top-p value
    - `vllm.sampling.top_k` - Top-k value
    - `vllm.sampling.tokens_sampled` - Number of tokens sampled
    - `vllm.sampling.duration_ms` - Sampling time
  - **Purpose**: Track sampling overhead

**5. `vllm.kv_cache.transfer`** (for disaggregated serving)
- **Location**: KV cache connector operations when using disaggregation
- **Kind**: `trace.SpanKindClient` (for send) or `trace.SpanKindServer` (for receive)
- **Attributes**:
  - `vllm.kv_transfer.direction` - send/receive
  - `vllm.kv_transfer.protocol` - rdma/tcp/nixl
  - `vllm.kv_transfer.blocks` - Number of blocks transferred
  - `vllm.kv_transfer.bytes` - Bytes transferred
  - `vllm.kv_transfer.duration_ms` - Transfer time
  - `vllm.kv_transfer.source_pod` - Source pod name
  - `vllm.kv_transfer.dest_pod` - Destination pod name
  - `vllm.kv_transfer.bandwidth_mbps` - Transfer bandwidth
- **Purpose**: Critical for diagnosing disaggregated serving performance

**6. `vllm.output.process_batch`**
- **Location**: `vllm/v1/engine/output_processor.py:383` - process_outputs method
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `vllm.outputs.processed` - Number of outputs processed
  - `vllm.outputs.finished` - Number of finished requests
  - `vllm.outputs.duration_ms` - Processing time
- **Purpose**: Measure output processing overhead
- **Child Spans**:

  **6a. `vllm.detokenizer.update`**
  - **Location**: Detokenizer update per request
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `vllm.tokens.new` - New tokens to detokenize
    - `vllm.stop_string.detected` - Whether stop string detected
    - `vllm.detokenize.duration_ms` - Detokenization time
  - **Purpose**: Track detokenization overhead
  - **Note**: One span per request in batch

  **6b. `vllm.logprobs.compute`**
  - **Location**: Logprobs processor
  - **Kind**: `trace.SpanKindInternal`
  - **Attributes**:
    - `vllm.logprobs.prompt` - Whether computing prompt logprobs
    - `vllm.logprobs.sample` - Whether computing sample logprobs
    - `vllm.logprobs.duration_ms` - Computation time
  - **Purpose**: Track logprobs computation overhead

**Implementation Priority**:
1. High: Enhance `vllm.engine.request`, `vllm.scheduler.schedule`, `vllm.executor.execute_model`
2. Medium: Scheduler child spans, `vllm.output.process_batch`, `vllm.processor.process_inputs`
3. Low: Worker/model/sampler child spans, detokenizer/logprobs spans
4. Future: `vllm.kv_cache.transfer` (when disaggregation is deployed)

---

### **`llm-d-kv-cache-manager`**

**Component Status**: The KV cache manager provides a global view of KV cache states across pods for intelligent routing decisions.

**Current Tracing State**: No existing tracing infrastructure identified.

**Proposed Manual Instrumentation**:

#### Span: `kvcache.manager.get_scores`
- **Location**: Main GetPodScores RPC handler
- **Kind**: `trace.SpanKindServer`
- **Attributes**:
  - `gen_ai.request.model` - Model identifier
  - `kvcache.pod_count` - Number of pods evaluated
  - `kvcache.total_blocks_available` - Total blocks across all pods
  - `kvcache.duration_ms` - Total scoring time
- **Purpose**: Measure cache manager response time

#### Child Spans:

**1. `kvcache.storage.lookup`**
- **Location**: Storage backend lookup operations
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `kvcache.lookup.cache_hit` - Whether block found in storage
  - `kvcache.lookup.blocks_found` - Number of blocks found
  - `kvcache.storage.backend` - Storage type (filesystem/redis/etc)
  - `kvcache.lookup.duration_ms` - Lookup time
- **Purpose**: Track storage lookup performance

**2. `kvcache.scorer.compute`**
- **Location**: Scoring algorithm execution
- **Kind**: `trace.SpanKindInternal`
- **Attributes**:
  - `kvcache.scorer.algorithm` - Scoring algorithm used
  - `kvcache.hit_ratio.avg` - Average hit ratio across pods
  - `kvcache.hit_ratio.max` - Maximum hit ratio
  - `kvcache.score.distribution` - Score distribution (JSON)
  - `kvcache.scorer.duration_ms` - Scoring time
- **Purpose**: Understand scoring decisions

**Implementation Priority**: Medium (implement when KV cache manager is actively used)

---

### **`llm-d-routing-sidecar (P/D Proxy)`** - Transitional

**Component Status**: This component is planned for deprecation as P/D routing moves into vLLM.

**Recommendation**: **Minimal instrumentation** given transitional status. Focus on basic request/response spans only.

**Proposed Manual Instrumentation**:

#### Span: `pd_proxy.request`
- **Location**: Main HTTP handler
- **Kind**: `trace.SpanKindServer`
- **Attributes**:
  - `pd_proxy.disaggregation` - Whether disaggregation occurred
  - `pd_proxy.prefill_pod` - Prefill pod address (if disaggregated)
  - `pd_proxy.decode_pod` - Decode pod address
  - `http.method`, `http.status_code`
- **Purpose**: Basic visibility during transition period

**Implementation Priority**: Low (given deprecation plans)

---

### Trace Context Propagation

**Critical Integration Point**: Gateway → vLLM

The gateway **must inject trace context** into HTTP headers when proxying to vLLM backend pods:

```go
// In gateway when making HTTP request to vLLM
func proxyToBackend(ctx context.Context, req *http.Request) {
    // Inject trace context into outgoing request
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))

    // Make request...
}
```

vLLM already extracts trace context:
```python
# vLLM already does this in output_processor.py:499
trace_context = extract_trace_context(engine_core_output.trace_headers)
```

**W3C TraceContext Headers**:
- `traceparent`: `00-<trace-id>-<span-id>-<flags>`
- `tracestate`: Vendor-specific context (optional)

**Validation**: End-to-end traces should show:
1. `gateway.request` (parent)
2. `gateway.backend.proxy` (child, kind=CLIENT)
3. `vllm.engine.request` (child of gateway.request via trace context)

---

### Sampling Strategy

**Recommended Approach**: **Parent-Based Sampling**

- Respect upstream sampling decisions (if llm-d is called by traced services)
- Allow independent sampling for llm-d-initiated operations
- Default sampling rate: **10%** (configurable via `OTEL_TRACES_SAMPLER_ARG`)

**Rationale**:
- Maintains trace continuity with upstream services
- Reduces trace volume and storage costs
- Still provides statistical visibility into performance
- Can increase sampling for specific requests via headers

**Configuration**:
```bash
# Gateway
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1

# vLLM - inherits from gateway via trace context
```

**Head-Based Sampling**: Sampling decision made at trace entry (gateway), propagated to all components.

---

### Enabling Distributed Tracing

Each component requires **explicit trace initialization** in code:

#### Gateway (Go)
```go
// Already implemented in pkg/common/telemetry.go:46
func InitTracing(ctx context.Context, logger logr.Logger) error {
    // Creates TracerProvider with OTLP exporter
    // Sets up W3C propagation
    // Configures sampling
}

// Called from cmd/epp/runner/runner.go:166-170
if *tracing {
    err := common.InitTracing(ctx, setupLog)
    if err != nil {
        return err
    }
}
```

**Configuration**:
```bash
OTEL_SERVICE_NAME=gateway-api-inference-extension
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_TRACES_EXPORTER=otlp  # or "console" for debugging
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1
```

#### vLLM (Python)
```python
# Already implemented in async_llm.py:134-138
if self.observability_config.otlp_traces_endpoint is not None:
    tracer = init_tracer(
        "vllm.llm_engine",
        self.observability_config.otlp_traces_endpoint
    )
    self.output_processor.tracer = tracer
```

**Configuration**:
```python
# Via VllmConfig
vllm_config.observability_config.otlp_traces_endpoint = "http://otel-collector:4317"
```

**Deployment**:
- Deploy OpenTelemetry Collector as sidecar or DaemonSet
- Collector receives traces via OTLP gRPC (port 4317)
- Collector exports to backend (Jaeger, Tempo, Datadog, etc.)

---

### Semantic Conventions and Attributes

**OpenTelemetry GenAI Semantic Conventions** (where applicable):
- `gen_ai.request.model` - Model identifier
- `gen_ai.request.id` - Request ID
- `gen_ai.usage.prompt_tokens` - Prompt tokens
- `gen_ai.usage.completion_tokens` - Completion tokens
- `gen_ai.latency.*` - Latency metrics
- `gen_ai.request.*` - Request parameters (temperature, top_p, etc.)

**llm-d Custom Attributes**:
- Namespace: `llm_d.*` for llm-d-specific attributes
- Component namespace: `vllm.*`, `kvcache.*`, etc.
- Consistent naming: lowercase with underscores

**Performance Considerations**:
- Avoid high-cardinality attributes (e.g., full request payloads)
- Use span events for point-in-time occurrences
- Sample high-frequency spans (per-iteration operations) more aggressively
- Record errors: `span.RecordError(err)`, `span.SetStatus(codes.Error, msg)`

---

## Implementation Approach

### Phased Rollout

#### Phase 1: Core Request Lifecycle (High Priority)

**Objective**: Establish end-to-end tracing with basic visibility

**Gateway**:
- `gateway.request` - Top-level request span with context extraction
- `gateway.director.handle_request` - Admission control
- `gateway.scheduler.schedule` - Pod selection (without child spans initially)
- `gateway.backend.proxy` - Backend call with **trace context injection**
- `gateway.response.process` - Response handling

**vLLM**:
- Enhance existing `vllm.engine.request` span with additional attributes
- `vllm.scheduler.schedule` - Batch scheduling overview
- `vllm.executor.execute_model` - Model execution overview

**Deliverables**:
- ✅ End-to-end trace from gateway to vLLM
- ✅ Basic latency breakdown (admission, scheduling, execution, response)
- ✅ Error tracking across components
- ✅ Trace context propagation verified

**Success Criteria**:
- Can view complete trace in Jaeger/Tempo showing gateway → vLLM flow
- Latency attribution identifies bottleneck component
- Errors propagate with proper status codes

---

#### Phase 2: Detailed Decision Visibility (Medium Priority)

**Objective**: Expose internal decision-making and optimize observability

**Gateway**:
- `gateway.scheduler.filter` - Individual filter execution
- `gateway.scheduler.score` - Individual scorer execution
- `gateway.scheduler.pick` - Selection logic
- `gateway.metrics.collect` - Metrics scraping (with aggressive sampling)
- `gateway.flowcontrol.*` - Flow control spans (if feature enabled)

**vLLM**:
- `vllm.scheduler.admit_requests` - Admission decisions
- `vllm.scheduler.allocate_blocks` - KV cache allocation
- `vllm.scheduler.create_batch` - Batch formation
- `vllm.output.process_batch` - Output processing
- `vllm.detokenizer.update` - Detokenization (sampled)
- `vllm.processor.process_inputs` - Input processing

**Deliverables**:
- ✅ Scheduler decision visibility (which filters/scorers ran, scores assigned)
- ✅ KV cache operation tracking (hits, misses, allocations)
- ✅ Batch composition metrics
- ✅ Token generation performance

**Success Criteria**:
- Can debug why specific pod was selected (filter/scorer details)
- Can measure prefix caching effectiveness
- Can identify slow schedulers or scorers

---

#### Phase 3: Fine-Grained Performance (Lower Priority)

**Objective**: Deep performance profiling and optimization validation

**Gateway**:
- `gateway.request.headers`, `gateway.request.body` - Request parsing
- Additional attribute richness (score distributions, weights)

**vLLM**:
- `vllm.worker.prepare_inputs` - Input preparation
- `vllm.model.forward` - Model forward pass (heavily sampled)
- `vllm.sampler.sample` - Token sampling
- `vllm.logprobs.compute` - Logprobs computation
- `vllm.kv_cache.transfer` - KV cache transfers (disaggregated serving)

**KV Cache Manager**:
- `kvcache.manager.get_scores` - Full instrumentation
- `kvcache.storage.lookup`, `kvcache.scorer.compute` - Child spans

**Deliverables**:
- ✅ Per-iteration performance profiling
- ✅ Disaggregated serving trace visibility
- ✅ Fine-grained bottleneck identification

**Success Criteria**:
- Can identify slow model layers or attention operations
- Can measure KV transfer bandwidth in disaggregated mode
- Can optimize based on per-component timing data

---

### Code Implementation Patterns

#### Gateway (Go) - Creating Spans

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/trace"
)

// Get global tracer (initialized in telemetry.go)
var tracer = otel.Tracer("gateway-api-inference-extension")

// Example: gateway.scheduler.schedule span
func (s *Scheduler) Schedule(ctx context.Context, req *SchedulingRequest) (*Pod, error) {
    ctx, span := tracer.Start(ctx, "gateway.scheduler.schedule",
        trace.WithSpanKind(trace.SpanKindInternal),
    )
    defer span.End()

    // Set initial attributes
    span.SetAttributes(
        attribute.String("scheduler.policy", s.config.Policy),
        attribute.Int("candidates.initial", len(candidates)),
    )

    // Schedule logic...
    pod, err := s.selectPod(ctx, candidates)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }

    // Set result attributes
    span.SetAttributes(
        attribute.String("selected.pod_name", pod.Name),
        attribute.String("selected.pod_ip", pod.IP),
        attribute.Float64("selected.score", pod.Score),
    )

    span.SetStatus(codes.Ok, "")
    return pod, nil
}

// Example: Child span for filter
func (f *Filter) Execute(ctx context.Context, pods []*Pod) ([]*Pod, error) {
    ctx, span := tracer.Start(ctx, "gateway.scheduler.filter")
    defer span.End()

    span.SetAttributes(
        attribute.String("filter.name", f.Name),
        attribute.String("filter.type", f.Type),
    )

    filtered := f.filter(pods)

    span.SetAttributes(
        attribute.Int("filter.rejected_count", len(pods) - len(filtered)),
    )

    return filtered, nil
}
```

#### vLLM (Python) - Creating Spans

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode

# Tracer already initialized in async_llm.py
# Access via self.output_processor.tracer

# Example: vllm.scheduler.schedule span
def schedule(self) -> SchedulerOutput:
    if self.tracer:
        with self.tracer.start_as_current_span(
            "vllm.scheduler.schedule",
            kind=trace.SpanKind.INTERNAL
        ) as span:
            span.set_attribute("vllm.scheduler.type", self.__class__.__name__)
            span.set_attribute("vllm.scheduler.iteration", self.iteration)

            try:
                output = self._schedule_internal()

                # Set result attributes
                span.set_attribute("vllm.batch.phase", output.phase)
                span.set_attribute("vllm.batch.total_tokens", output.total_tokens)
                span.set_attribute("vllm.kv_cache.blocks_allocated", output.blocks_allocated)

                span.set_status(Status(StatusCode.OK))
                return output

            except Exception as e:
                span.record_exception(e)
                span.set_status(Status(StatusCode.ERROR, str(e)))
                raise
    else:
        return self._schedule_internal()

# Example: Child span
def admit_requests(self, ctx) -> AdmissionResult:
    if self.tracer:
        with self.tracer.start_as_current_span(
            "vllm.scheduler.admit_requests"
        ) as span:
            span.set_attribute("vllm.requests.waiting", len(self.waiting))

            result = self._admit_internal()

            span.set_attribute("vllm.requests.admitted", len(result.admitted))
            span.set_attribute("vllm.requests.preempted", len(result.preempted))

            return result
    else:
        return self._admit_internal()
```

---

## Alternatives

### Auto-Instrumentation via Agents

**Description**: Use OpenTelemetry auto-instrumentation agents (Go, Python) to automatically instrument HTTP/gRPC servers and clients without code changes.

**Pros**:
- Zero code changes required
- Automatic HTTP/gRPC span creation
- Standardized instrumentation

**Cons**:
- ❌ **No custom spans** - Cannot instrument llm-d-specific operations (scheduling, KV cache, etc.)
- ❌ **Generic attributes** - Lacks llm-d domain knowledge (filter decisions, scores, cache hits)
- ❌ **Limited control** - Cannot control span granularity or sampling per operation
- ❌ **Security risk** - May capture HTTP bodies by default, exposing sensitive prompts
- ❌ **No value for debugging** - Cannot see internal decision-making processes

**Decision**: ❌ Rejected - Auto-instrumentation provides only surface-level visibility. llm-d requires deep observability into scheduling, caching, and model execution decisions that auto-instrumentation cannot provide.

---

### Third-Party APM Solutions (Datadog, New Relic, etc.)

**Description**: Use commercial APM agents instead of OpenTelemetry.

**Pros**:
- Integrated dashboards and alerting
- Automatic instrumentation
- Support teams

**Cons**:
- ❌ Vendor lock-in
- ❌ May lack GenAI semantic conventions
- ❌ Higher cost
- ❌ Still cannot provide llm-d-specific custom spans without manual instrumentation

**Decision**: ❌ Rejected - Most APM vendors now support OpenTelemetry anyway. Manual OTel instrumentation provides vendor neutrality and precise control over what's traced.

---

### Metrics-Only Approach

**Description**: Use Prometheus metrics instead of distributed tracing.

**Pros**:
- Lower overhead
- Mature ecosystem
- Simpler implementation

**Cons**:
- ❌ **No request-level visibility** - Cannot trace individual request flows
- ❌ **No cross-component correlation** - Cannot see how request moved through system
- ❌ **Limited debugging** - Aggregate metrics don't explain why specific request failed
- ❌ **No latency attribution** - Cannot break down where time was spent per request

**Decision**: ❌ Rejected - Metrics and traces are complementary. Metrics provide aggregates, traces provide per-request debugging capability. Both are needed.

---

### Langfuse for LLM Observability

**Description**: Use Langfuse as an alternative or complement to OpenTelemetry for LLM-specific observability. Langfuse is an open-source LLM engineering platform providing observability, metrics, prompt management, evaluations, and datasets.

#### Langfuse Overview

**What Langfuse Provides**:
- **LLM-Specific Tracing**: Purpose-built for LLM applications with native concepts like generations, spans, and events
- **Prompt Management**: Version control and tracking for prompts, linking traces to specific prompt versions
- **Cost Tracking**: Automatic token and cost calculation for major LLM providers (OpenAI, Anthropic, Google, etc.)
- **Quality Metrics**: User feedback collection, scoring, and evaluation frameworks
- **Rich UI**: Purpose-built dashboard for LLM traces with prompt/completion viewing, token counts, and latency breakdowns
- **Prompt Playground**: Test and iterate on prompts with real production data

**What Langfuse Does NOT Provide (vs. Infrastructure Observability)**:
- ❌ Infrastructure-level tracing (HTTP servers, databases, message queues)
- ❌ Kubernetes/container observability
- ❌ Network-level distributed tracing
- ❌ General-purpose APM capabilities

#### Integration Options

**Option 1: Langfuse Native SDK Only** ❌ Not Recommended for llm-d

Use Langfuse Python/JS SDKs with decorators (`@observe`) or context managers:

```python
from langfuse.decorators import observe

@observe()
def my_llm_function():
    response = openai.chat.completions.create(...)
    return response
```

**Pros**:
- Simple decorator-based instrumentation
- Rich LLM-specific features out of the box
- Beautiful UI for viewing prompts and completions

**Cons**:
- ❌ **Cannot trace llm-d infrastructure components** (gateway, scheduler written in Go)
- ❌ **Limited to Python/JS** - Gateway is Go-based
- ❌ **Separate from infrastructure traces** - No unified view of gateway + vLLM
- ❌ **Requires prompt/completion access** - Security concerns for llm-d infrastructure layer

**Decision for llm-d**: ❌ Rejected - Langfuse native SDK cannot instrument Go components (gateway, scheduler) and doesn't provide infrastructure-level visibility we need.

---

**Option 2: OpenTelemetry → Langfuse (Recommended Hybrid Approach)** ✅ Recommended

Use OpenTelemetry for all instrumentation, export traces to Langfuse via OTLP endpoint. This is your current setup!

**Architecture**:
```
llm-d Components → OpenTelemetry SDK → OTLP Export → Langfuse (receives on /api/public/otel)
                                              ↓
                                    Also export to Jaeger/Tempo/etc. (optional)
```

**Implementation**:
```bash
# Gateway & vLLM both export to Langfuse
OTEL_EXPORTER_OTLP_ENDPOINT=https://cloud.langfuse.com/api/public/otel
OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic $(echo -n 'pk-lf-xxx:sk-lf-xxx' | base64)"
```

**What You Get**:
- ✅ **Unified instrumentation**: Same OTel spans work for both infrastructure and LLM observability
- ✅ **Langfuse LLM features**: Cost tracking, prompt management, quality metrics, rich UI
- ✅ **Infrastructure visibility**: All llm-d custom spans (scheduler, filters, cache) visible in Langfuse
- ✅ **Multi-backend export**: Can send to Langfuse AND Jaeger/Tempo simultaneously via OTel Collector
- ✅ **GenAI semantic conventions**: Langfuse supports OTel GenAI attributes (`gen_ai.usage.prompt_tokens`, etc.)
- ✅ **Language agnostic**: Works for Go (gateway) and Python (vLLM)

**Langfuse OTel Support (as of Feb 2025)**:
- Accepts spans on `/api/public/otel` OTLP endpoint
- Maps OTel spans to Langfuse data model (spans → observations, generations)
- Supports GenAI semantic conventions
- Extracts token counts, costs, model info from OTel attributes
- New OTel-native Langfuse SDK v3 available (thin wrapper on OTel SDK)

**Pros**:
- ✅ Best of both worlds: Infrastructure tracing + LLM-specific features
- ✅ Single instrumentation codebase (OTel)
- ✅ Can export to multiple backends (Langfuse + traditional APM)
- ✅ Langfuse UI for LLM-specific analysis
- ✅ Traditional trace viewers (Jaeger) for infrastructure debugging
- ✅ No vendor lock-in (standard OTel)

**Cons**:
- Need to set OTel attributes correctly for Langfuse to extract LLM metadata
- Langfuse UI may show infrastructure spans alongside LLM spans (can filter)
- Requires understanding both OTel and Langfuse data models

**Decision for llm-d**: ✅ **RECOMMENDED** - This is the ideal approach and what you're already using!

---

#### Where to Add Langfuse-Specific Instrumentation in llm-d

Given the recommended OpenTelemetry → Langfuse approach, here's where to add **LLM-specific attributes** to make Langfuse most valuable:

**1. vLLM Layer (Highest Value)** ⭐⭐⭐

**Location**: `vllm/v1/engine/output_processor.py` - Enhance existing `llm_request` span

**Add Langfuse-Friendly Attributes**:
```python
# GenAI semantic conventions (already proposed, Langfuse will parse these)
span.set_attribute("gen_ai.request.model", model_name)
span.set_attribute("gen_ai.usage.prompt_tokens", prompt_tokens)
span.set_attribute("gen_ai.usage.completion_tokens", completion_tokens)
span.set_attribute("gen_ai.request.temperature", temperature)
span.set_attribute("gen_ai.request.top_p", top_p)
span.set_attribute("gen_ai.request.max_tokens", max_tokens)

# Langfuse-specific for richer features (optional)
span.set_attribute("langfuse.observation.name", "vllm-generation")
span.set_attribute("langfuse.observation.type", "generation")  # vs "span" or "event"

# Cost tracking (if you calculate it)
span.set_attribute("gen_ai.usage.cost", calculated_cost_usd)

# Prompt management (if using Langfuse prompt versioning)
# span.set_attribute("langfuse.prompt.name", "system-prompt-v2")
# span.set_attribute("langfuse.prompt.version", 2)
```

**Why Here**: vLLM has access to all LLM-specific metadata (tokens, model, params). Langfuse will automatically:
- Calculate costs based on model + token counts
- Display generations with rich metadata
- Track token usage over time
- Enable filtering by model, cost, token range

**Security Note**: Do NOT add `gen_ai.prompt.0.content` or `gen_ai.completion.0.content` attributes! Keep metadata-only approach.

---

**2. Gateway Layer (Medium Value)** ⭐⭐

**Location**: `gateway.request` span in `pkg/epp/handlers/server.go`

**Add High-Level Metadata**:
```go
span.SetAttributes(
    attribute.String("gen_ai.request.model", incomingModelName),
    attribute.String("langfuse.trace.name", "llm-inference-request"),
    attribute.String("langfuse.trace.user_id", extractedUserID),  // if available
    attribute.String("langfuse.trace.session_id", extractedSessionID),  // if available
    attribute.String("langfuse.trace.tags", "gateway,production"),
)
```

**Why Here**: Gateway is the entry point and may have user/session context. Langfuse can:
- Group traces by user/session
- Filter by tags
- Track user-level metrics

**Security Note**: Only add user_id/session_id if they don't contain PII or you have appropriate data governance.

---

**3. Application Layer (Outside llm-d Scope)** ⭐⭐⭐⭐

**Recommendation**: If you have an application layer BEFORE llm-d (e.g., RAG pipeline, agent framework, API gateway), that's the BEST place for Langfuse instrumentation with prompt/completion tracking.

**Architecture**:
```
Application Layer (RAG/Agent) → llm-d Gateway → vLLM
     ↓ (can use Langfuse SDK)      ↓ (OTel)     ↓ (OTel)
     Langfuse traces with          Infrastructure   LLM execution
     prompts/completions           observability    observability
```

**Why**: Application layer:
- Has access to actual prompts/completions (can log if appropriate)
- Can use Langfuse prompt management features
- Can collect user feedback and quality metrics
- Separate security boundary from infrastructure

**Implementation**: Use Langfuse decorators or OTel with full GenAI attributes including content:
```python
# At application layer (if security policy allows)
span.set_attribute("gen_ai.prompt.0.role", "user")
span.set_attribute("gen_ai.prompt.0.content", user_prompt)  # OK here if authorized
span.set_attribute("gen_ai.completion.0.role", "assistant")
span.set_attribute("gen_ai.completion.0.content", completion)  # OK here if authorized
```

---

#### Using OpenTelemetry Collector as Middle-Man

If you want to send OTel traces to BOTH Langfuse and traditional backends:

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 10s

exporters:
  # Export to Langfuse
  otlphttp/langfuse:
    endpoint: "https://cloud.langfuse.com/api/public/otel"
    headers:
      Authorization: "Basic ${LANGFUSE_AUTH}"

  # Also export to Jaeger for infrastructure debugging
  jaeger:
    endpoint: "jaeger-collector:14250"
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/langfuse, jaeger]  # Dual export
```

**Benefits**:
- Langfuse for LLM-specific analysis (costs, tokens, quality)
- Jaeger for infrastructure debugging (network, latency, errors)
- Single OTel instrumentation codebase
- Filter/route spans based on attributes if needed

---

#### Langfuse Features Unlocked with OTel Integration

With OpenTelemetry → Langfuse, you get:

**1. Cost Tracking**
- Automatic cost calculation based on `gen_ai.usage.*_tokens` + model name
- Cost dashboard and alerts
- Cost attribution by user/session/tag

**2. Token Analytics**
- Token usage over time
- Prompt vs completion token distribution
- Identify expensive requests

**3. Latency Analysis**
- TTFT distribution (from `gen_ai.latency.time_to_first_token`)
- E2E latency by model, user, tag
- Identify slow requests

**4. Prompt Management** (if you adopt it)
- Version prompts in Langfuse UI
- Link traces to specific prompt versions
- A/B test prompts

**5. Quality Metrics** (requires additional integration)
- Collect user feedback on completions
- Calculate quality scores
- Identify hallucinations or errors

**6. LLM-Specific UI**
- View traces with LLM context (not just generic spans)
- Filter by model, token count, cost
- Drill down into specific generations

---

#### Recommendation Summary

**For llm-d Infrastructure**:
✅ **Use OpenTelemetry with manual instrumentation** as proposed
✅ **Export to Langfuse via OTLP** for LLM-specific observability
✅ **Add GenAI semantic convention attributes** in vLLM spans
✅ **Optionally dual-export** to Jaeger/Tempo for infrastructure debugging

**Where to Instrument**:
- **Gateway**: Metadata only (model, user/session if available, tags)
- **vLLM**: Rich LLM metadata (tokens, model, params, latency) - DO NOT include prompts/completions
- **Application Layer** (outside llm-d): Prompts/completions if security policy allows

**Security**:
- llm-d infrastructure: **Metadata-only** tracing (no prompts/completions)
- Application layer: May include prompts/completions if appropriate

**Cost**:
- Langfuse Cloud: Free tier available, paid plans based on trace volume
- Self-hosted Langfuse: Open source, free to run

**Decision**: ✅ **COMPLEMENT, NOT REPLACE** - Use OpenTelemetry for infrastructure observability + Langfuse as an LLM-aware backend. This is what you're already doing, and it's the optimal approach!

---

## Security Considerations

### Data Sensitivity in LLM Inference

LLM inference workloads process highly sensitive data including proprietary prompts, personal information, confidential business data, and intellectual property.
LLM queries and responses frequently contain:

- Confidential communications
- Personal identifiable information (PII) and regulated data
- Proprietary code, algorithms, and technical specifications
- Sensitive healthcare, financial, or legal information

This sensitive data requires specialized handling in observability systems to prevent inadvertent exposure through trace data.

### Tracing Security Model

This proposal implements a **metadata-only tracing approach** that provides operational visibility while protecting sensitive data:

**What is Captured**:
- Request timing and performance metrics (TTFT, ITL, total latency)
- Model identifiers, component routing decisions, and operational metadata
- Error classifications, timeout events, and success/failure states
- Component-to-component communication patterns and trace context
- KV cache hit ratios, pod selection metadata, scheduling decisions
- Token **counts** (prompt_tokens, completion_tokens) - NOT actual tokens

**What is Explicitly Excluded**:
- ❌ Request payloads (prompts, user inputs, system messages)
- ❌ Response content (generated text, completions, model outputs)
- ❌ Intermediate processing content (embeddings, vector representations)
- ❌ Actual tokens or token IDs
- ❌ Any form of request/response body content
- ❌ HTTP/gRPC headers that may contain sensitive data (except trace context headers)

### Security Goals

* **Data Privacy by Design**: Ensure no sensitive request/response content is captured in trace data, regardless of trace export destination or retention policies.

* **Operational Security**: Provide sufficient observability for performance optimization and debugging without compromising data confidentiality or regulatory compliance.

* **Secure Configuration**: Enable tracing through well-defined, auditable configuration paths that maintain security boundaries across llm-d components.

* **Manual Control**: Explicit span creation ensures developers consciously decide what data enters spans, preventing accidental exposure.

### Implementation Security Measures

**Manual Instrumentation Security Benefits**:
- ✅ Explicit control over span attributes - no accidental body capture
- ✅ Code review process ensures no sensitive data in attributes
- ✅ No dependency on agent configuration for security
- ✅ Developer awareness of what's being traced

**Span Attribute Filtering**:
```go
// ✅ SAFE: Metadata only
span.SetAttributes(
    attribute.Int("gen_ai.usage.prompt_tokens", len(tokens)),
    attribute.String("gen_ai.request.model", "llama-2-70b"),
)

// ❌ UNSAFE: Sensitive content
span.SetAttributes(
    attribute.String("request.prompt", userPrompt), // NEVER DO THIS
    attribute.StringSlice("response.tokens", generatedTokens), // NEVER DO THIS
)
```

**Context Propagation Security**:
- Trace context uses W3C TraceContext headers (`traceparent`, `tracestate`)
- Contains only trace/span IDs and flags - no business data
- Safe to propagate across trust boundaries

**HTTP Instrumentation Security**:
```go
// When using otelhttp for HTTP server instrumentation
handler := otelhttp.NewHandler(
    http.HandlerFunc(yourHandler),
    "gateway.request",
    // DO NOT use WithMessageEvents - it captures HTTP bodies
)
```

**Export Security**:
- Use TLS for OTLP exporter: `OTEL_EXPORTER_OTLP_ENDPOINT=https://collector:4317`
- Implement authentication for trace collector
- Configure trace retention policies aligned with data governance
- Treat trace data as operationally sensitive metadata

**Testing Security**:
- Code review checklist: No span attributes contain user content
- Security testing: Validate no prompts/completions in exported traces
- Regular audits of span attributes in production

---

## Testing and Validation

### Unit Testing

**Gateway (Go)**:
```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/otel/sdk/trace/tracetest"
)

func TestSchedulerSpan(t *testing.T) {
    // Create in-memory span recorder
    sr := tracetest.NewSpanRecorder()
    provider := trace.NewTracerProvider(trace.WithSpanProcessor(sr))
    otel.SetTracerProvider(provider)

    // Run operation
    scheduler.Schedule(ctx, request)

    // Verify spans
    spans := sr.Ended()
    require.Len(t, spans, 1)
    assert.Equal(t, "gateway.scheduler.schedule", spans[0].Name())

    // Verify attributes
    attrs := spans[0].Attributes()
    assert.Contains(t, attrs, attribute.String("scheduler.policy", "weighted"))
}
```

**vLLM (Python)**:
```python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export.in_memory_span_exporter import InMemorySpanExporter

def test_scheduler_span():
    # Setup in-memory exporter
    exporter = InMemorySpanExporter()
    provider = TracerProvider()
    provider.add_span_processor(SimpleSpanProcessor(exporter))

    scheduler.tracer = provider.get_tracer("test")

    # Run operation
    scheduler.schedule()

    # Verify spans
    spans = exporter.get_finished_spans()
    assert len(spans) == 1
    assert spans[0].name == "vllm.scheduler.schedule"
    assert "vllm.batch.total_tokens" in spans[0].attributes
```

### Integration Testing

**End-to-End Trace Validation**:
1. Deploy llm-d with Jaeger backend
2. Send test inference request
3. Query Jaeger API for trace
4. Validate trace structure:
   - Root span: `gateway.request`
   - Child span: `gateway.scheduler.schedule`
   - Child span: `gateway.backend.proxy` (CLIENT)
   - Child span: `vllm.engine.request` (child of gateway.request)
5. Validate trace context propagation (same trace_id)
6. Validate span attributes present

**Sampling Validation**:
1. Configure 100% sampling
2. Send N requests
3. Verify N traces in backend
4. Configure 10% sampling
5. Send 1000 requests
6. Verify ~100 traces (statistical)

### Performance Testing

**Overhead Measurement**:
1. Baseline: Measure throughput/latency with tracing disabled
2. Measure with tracing enabled (10% sampling)
3. Measure with tracing enabled (100% sampling)
4. Target: <5% latency overhead at 10% sampling

**Load Testing**:
1. Run high-load test (1000 RPS) with tracing enabled
2. Monitor CPU/memory usage
3. Verify no degradation beyond acceptable threshold
4. Monitor trace backend (collector, storage) capacity

### Production Rollout

**Phase 1: Canary (1% sampling)**
- Deploy to 10% of traffic
- Monitor for performance impact
- Validate trace data quality

**Phase 2: Gradual Increase (10% sampling)**
- Deploy to 50% of traffic
- Increase sampling to 10%
- Monitor trace volume and storage costs

**Phase 3: Full Rollout (10% sampling)**
- Deploy to 100% of traffic
- Tune sampling based on trace value and cost

---

## Metrics vs. Traces vs. Logs

**Use Traces for**:
- Request-scoped operations (single request flow)
- Latency breakdown and bottleneck identification
- Error propagation and debugging
- Cross-service correlation
- Understanding "why this request was slow"

**Use Metrics for**:
- Aggregated statistics (throughput, error rates)
- Resource utilization (KV cache, GPU memory)
- SLO monitoring (P50, P95, P99 latencies)
- Alerting thresholds
- Understanding "is the system healthy"

**Use Logs for**:
- Detailed error messages and stack traces
- Debugging information
- Audit trails
- Events without request context
- Understanding "what happened in this component"

**Correlation**:
- Include `trace_id` in log messages for correlation
- Include `request_id` in both traces and logs
- Use trace context in metrics labels (cardinality permitting)

---

## Success Criteria

This tracing implementation will be considered successful when:

1. ✅ **End-to-End Visibility**: Can view complete request flow from gateway to vLLM in trace UI
2. ✅ **Latency Attribution**: Can identify which component/operation is bottleneck for slow requests
3. ✅ **Decision Transparency**: Can see why specific pod was selected (filter/scorer details)
4. ✅ **Error Debugging**: Can trace failed requests through system and identify failure point
5. ✅ **Optimization Validation**: Can measure impact of KV cache hits, P/D disaggregation on latency
6. ✅ **Performance Acceptable**: <5% latency overhead at 10% sampling
7. ✅ **Security Compliant**: No prompts or completions in trace data (validated via audit)
8. ✅ **Developer Adoption**: Component owners can add new spans without tracing expertise

---

## Future Enhancements

### Langfuse Integration

**Langfuse** is an LLM observability platform focused on prompt engineering, quality, and cost tracking.

**Integration Strategy**:
- **OpenTelemetry**: Infrastructure/performance observability (this proposal)
- **Langfuse**: LLM application observability (prompts, quality, cost)

**Langfuse Capabilities**:
- Prompt versioning and A/B testing
- User feedback collection
- Cost tracking per prompt
- Quality metrics (hallucination detection, etc.)

**Correlation**:
- Use same `request_id` in both OTel and Langfuse
- OTel provides performance context, Langfuse provides quality context

**Implementation Consideration**:
- Langfuse may need access to prompts/completions (privacy implications)
- Deploy Langfuse instrumentation separately from OTel
- Gateway or application layer (not llm-d) may be better suited for Langfuse

### Span Links

**Use Case**: When llm-d receives requests from multiple upstream services, span links can maintain correlation
without breaking parent-based sampling decisions.

**Implementation**:
```go
ctx, span := tracer.Start(ctx, "gateway.request",
    trace.WithLinks(trace.LinkFromContext(upstreamCtx)),
)
```

### Trace Sampling by Request Type

**Use Case**: Sample expensive models (Llama-70B) at higher rate than cheap models (Llama-7B)

**Implementation**: Custom sampler based on model name attribute

### Real-Time Trace Analytics

**Use Case**: Alert on trace patterns (e.g., high error rate for specific model)

**Implementation**: Stream traces to analytics platform (OpenSearch, ClickHouse)

---

## Appendix: Environment Variables

### Gateway Extension
```bash
OTEL_SERVICE_NAME=gateway-api-inference-extension
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_TRACES_EXPORTER=otlp  # or "console" for debugging
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1  # 10% sampling
```

### vLLM
```python
# Set in VllmConfig.observability_config
observability_config.otlp_traces_endpoint = "http://otel-collector:4317"
```

### OpenTelemetry Collector (Helm values)
```yaml
config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
  processors:
    batch:
      timeout: 10s
      send_batch_size: 1024
  exporters:
    jaeger:
      endpoint: jaeger-collector:14250
      tls:
        insecure: true
  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [batch]
        exporters: [jaeger]
```

---

## Appendix: Key File Locations

### Gateway Extension
- OTel initialization: `pkg/common/telemetry.go:46-140`
- EPP server: `pkg/epp/handlers/server.go`
- Director: `pkg/epp/requestcontrol/director.go`
- Scheduler: `pkg/epp/scheduling/scheduler.go`
- Filters: `pkg/epp/scheduling/framework/plugins/*/filter/`
- Scorers: `pkg/epp/scheduling/framework/plugins/scorer/`
- Pickers: `pkg/epp/scheduling/framework/plugins/picker/`
- Flow control: `pkg/epp/flowcontrol/controller/controller.go`

### vLLM v1
- OTel initialization: `vllm/v1/engine/async_llm.py:134-138`, `vllm/v1/engine/llm_engine.py:115-119`
- Existing span: `vllm/v1/engine/output_processor.py:488-550`
- Engine core: `vllm/v1/engine/core.py`
- Scheduler: `vllm/v1/core/sched/scheduler.py`
- Executor: `vllm/v1/executor/*.py`
- Output processor: `vllm/v1/engine/output_processor.py`
- Processor: `vllm/v1/engine/processor.py`

---

## Appendix: Example Jaeger Queries

**View all gateway requests:**
```
service="gateway-api-inference-extension" AND operation="gateway.request"
```

**View end-to-end traces (gateway + vLLM):**
```
service="gateway-api-inference-extension" OR service="vllm.llm_engine"
```

**Find slow requests (>1s):**
```
duration > 1s
```

**Find requests with errors:**
```
error=true
```

**Find requests that hit specific filter:**
```
gateway.scheduler.filter.name="header-based-testing"
```

**Find requests with low KV cache hit ratio:**
```
vllm.kv_cache.hit_ratio < 0.5
```

---

## References

- OpenTelemetry Documentation: https://opentelemetry.io/docs/
- OpenTelemetry Go: https://opentelemetry.io/docs/languages/go/
- OpenTelemetry Python: https://opentelemetry.io/docs/languages/python/
- GenAI Semantic Conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- W3C TraceContext: https://www.w3.org/TR/trace-context/
- llm-d Architecture: https://github.com/llm-d/llm-d
- vLLM Documentation: https://docs.vllm.ai
- Gateway API Inference Extension: https://gateway-api-inference-extension.sigs.k8s.io/
