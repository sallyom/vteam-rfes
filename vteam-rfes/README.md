# vTeam RFEs - llm-d OpenTelemetry Tracing

This directory contains documentation and proposals for adding comprehensive OpenTelemetry tracing to the llm-d distributed inference stack.

## Documents

### 1. `llmd-tracing-proposal.md` ⭐
**Formal proposal for distributed tracing implementation**

- **Status**: Updated from original auto-instrumentation approach to **manual trace initialization**
- **Scope**: All llm-d components (gateway, vLLM, KV cache manager, routing sidecar)
- **Approach**: Explicit trace initialization with custom spans at strategic points
- **Key Changes from Original**:
  - ❌ No longer using auto-instrumentation
  - ✅ Manual OpenTelemetry SDK initialization per component
  - ✅ Detailed custom span specifications for each component
  - ✅ Precise control over attributes and sampling
  - ✅ Security by design (explicit span creation prevents accidental data exposure)
  - ✅ **NEW**: Comprehensive Langfuse integration strategy in Alternatives section

### 2. `otel-tracing-recommendations.md`
**Technical implementation guide with specific span recommendations**

- **Focus**: Practical implementation details for developers
- **Content**:
  - Current state analysis (what exists, what's missing)
  - Detailed span hierarchy and attributes per component
  - Code patterns (Go and Python)
  - Performance considerations
  - Testing strategies
  - Langfuse integration considerations

### 3. `LANGFUSE_INTEGRATION_SUMMARY.md` ⭐ NEW!
**Comprehensive guide for using Langfuse with OpenTelemetry**

- **Status**: Validates your current approach (OTel → Langfuse) as optimal!
- **Content**:
  - Why OpenTelemetry → Langfuse is the recommended approach
  - Specific GenAI semantic conventions to add for Langfuse features
  - Where to instrument (vLLM, gateway, application layer)
  - Configuration examples (direct export, dual export via collector)
  - Security considerations (metadata-only at infrastructure level)
  - Cost tracking, prompt management, quality metrics
  - **TL;DR**: Keep your current setup, just add more GenAI attributes!

## Key Findings

### Current State

**gateway-api-inference-extension (Go)**:
- ✅ OTel SDK fully initialized (`pkg/common/telemetry.go`)
- ❌ **Zero custom spans implemented**
- Infrastructure ready, just needs span creation

**vLLM v1 (Python)**:
- ✅ Tracer initialized when configured
- ✅ **One span**: `llm_request` with comprehensive attributes
- ❌ Only end-to-end span, no internal operation visibility

### Proposed Custom Spans

#### Gateway (15-20 spans)
**High Priority**:
- `gateway.request` (parent)
- `gateway.scheduler.schedule` + child spans (filter, score, pick)
- `gateway.director.handle_request`
- `gateway.backend.proxy` (with trace context injection)

**Medium Priority**:
- `gateway.response.process`
- `gateway.metrics.collect`
- Flow control spans (if feature enabled)

#### vLLM v1 (12-15 spans)
**High Priority**:
- Enhance `vllm.engine.request` (existing span)
- `vllm.scheduler.schedule` + child spans (admit, allocate, create_batch)
- `vllm.executor.execute_model`

**Medium Priority**:
- `vllm.output.process_batch` + child spans
- `vllm.processor.process_inputs`
- Worker/model/sampler spans (sampled)

**Future**:
- `vllm.kv_cache.transfer` (for disaggregated serving)

## Critical Integration Point

**Gateway → vLLM Trace Propagation**

The gateway **must inject** W3C TraceContext headers when proxying to vLLM:

```go
// In gateway backend proxy
otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))
```

vLLM already extracts trace context (implemented in `output_processor.py:499`)

## Implementation Phases

### Phase 1: Core Request Lifecycle
- Gateway: request, director, scheduler (overview), backend proxy, response
- vLLM: enhanced request span, scheduler overview, executor overview
- **Goal**: End-to-end trace visibility

### Phase 2: Detailed Decision Visibility
- Gateway: scheduler child spans (filter, score, pick), metrics, flow control
- vLLM: scheduler child spans, output processing, input processing
- **Goal**: Debug scheduling and batching decisions

### Phase 3: Fine-Grained Performance
- Gateway: request parsing, enhanced attributes
- vLLM: worker/model/sampler spans, KV cache transfers
- **Goal**: Performance profiling and optimization

## Langfuse Integration

**Status**: ✅ **Recommended as Complement to OpenTelemetry**

The proposal now includes a comprehensive Langfuse integration strategy:

**Approach**: OpenTelemetry → Langfuse (via OTLP endpoint)
- Single OTel instrumentation codebase
- Export to Langfuse for LLM-specific observability
- Optionally dual-export to Jaeger/Tempo for infrastructure debugging

**What Langfuse Adds**:
- 💰 Automatic cost tracking (tokens × model pricing)
- 📊 LLM-specific UI with rich metadata
- 📝 Prompt management and versioning
- ⭐ Quality metrics and user feedback
- 🔍 Filter by model, cost, token range

**Where to Add Langfuse-Friendly Attributes**:
1. **vLLM** (⭐⭐⭐ Highest value): GenAI semantic conventions for tokens, model, params
2. **Gateway** (⭐⭐ Medium value): User/session context, tags
3. **Application Layer** (⭐⭐⭐⭐ outside llm-d): Prompts/completions if security allows

**Key Point**: Use the SAME OpenTelemetry spans, just add GenAI semantic convention attributes that Langfuse will parse automatically!

## Security

**Metadata-Only Tracing**:
- ✅ Capture: latencies, token counts, model names, decisions, errors
- ❌ Never capture: prompts, completions, tokens, request/response bodies

**Manual Instrumentation Security Benefits**:
- Explicit control over span attributes
- Code review ensures no sensitive data
- No dependency on agent configuration

**Langfuse Security Note**:
- llm-d infrastructure: metadata only (no prompts/completions)
- Application layer (outside llm-d): may track prompts if appropriate

## Performance Targets

- **Overhead**: <5% latency increase at 10% sampling
- **Sampling**: Parent-based with configurable ratio (default 10%)
- **Volume**: Approximately 100 traces per 1000 requests at default sampling

## Getting Started

1. **Read the proposal**: `llmd-tracing-proposal.md` for full context
2. **Review recommendations**: `otel-tracing-recommendations.md` for implementation details
3. **Start with Phase 1**: Core request lifecycle spans
4. **Validate**: Ensure trace context propagation works end-to-end
5. **Iterate**: Add Phase 2 and 3 spans incrementally

## References

- **OpenTelemetry**: https://opentelemetry.io/docs/
- **GenAI Semantic Conventions**: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- **llm-d**: https://github.com/llm-d/llm-d
- **vLLM**: https://docs.vllm.ai

## Questions?

For questions or clarifications, refer to:
- The detailed proposals in this directory
- OpenTelemetry documentation
- llm-d architecture documentation

---

**Last Updated**: 2025-11-16
**Status**: Ready for implementation
**Next Steps**: Begin Phase 1 implementation in gateway-api-inference-extension
