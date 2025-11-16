# Langfuse Integration with llm-d OpenTelemetry Tracing

## TL;DR

✅ **Your current approach is optimal!** Continue using OpenTelemetry instrumentation and exporting to Langfuse via OTLP.

## What You're Already Doing Right

You mentioned you're "using Langfuse and OpenTelemetry in the same project and sending both to Langfuse backend & UI" - this is exactly the recommended approach!

**Architecture**:
```
llm-d Components → OpenTelemetry SDK → OTLP Export → Langfuse (/api/public/otel)
                                              ↓
                                    (optional) → Jaeger/Tempo
```

## Why This Works Perfectly

### 1. Best of Both Worlds

**OpenTelemetry Provides**:
- Infrastructure-level tracing (gateway, scheduler, routing)
- Cross-language support (Go gateway + Python vLLM)
- Vendor-neutral standard
- Deep component visibility with custom spans

**Langfuse Adds**:
- 💰 Automatic cost tracking (tokens × model pricing)
- 📊 LLM-specific UI with rich visualization
- 📝 Prompt management and versioning
- ⭐ Quality metrics and evaluation
- 🔍 Advanced filtering (model, cost, tokens)

### 2. Single Instrumentation Codebase

You write ONE set of OpenTelemetry spans with GenAI semantic conventions:

```python
# vLLM - these attributes work for both infrastructure and Langfuse
span.set_attribute("gen_ai.request.model", "llama-2-70b")
span.set_attribute("gen_ai.usage.prompt_tokens", 150)
span.set_attribute("gen_ai.usage.completion_tokens", 50)
span.set_attribute("gen_ai.request.temperature", 0.7)
```

**Langfuse automatically**:
- Calculates costs: `(150 input tokens × $0.0007) + (50 output × $0.0028) = $0.245`
- Displays in LLM-specific UI
- Enables filtering by model, cost, token range
- Links to prompt versions (if you use that feature)

### 3. Multi-Backend Export (Optional)

Use OpenTelemetry Collector to split traces:

```yaml
exporters:
  otlphttp/langfuse:
    endpoint: "https://cloud.langfuse.com/api/public/otel"
    headers:
      Authorization: "Basic ${LANGFUSE_AUTH}"

  jaeger:
    endpoint: "jaeger:14250"

service:
  pipelines:
    traces:
      exporters: [otlphttp/langfuse, jaeger]  # Dual export!
```

**Use Cases**:
- Langfuse: LLM cost analysis, quality tracking, prompt engineering
- Jaeger: Infrastructure debugging, network tracing, performance profiling

## Recommended Enhancements for llm-d

### Phase 1: vLLM Layer (Already Partially Implemented)

**Current State**: vLLM has one span with basic GenAI attributes

**Enhancement**: Add more GenAI semantic conventions for richer Langfuse features

```python
# vllm/v1/engine/output_processor.py - enhance existing llm_request span
span.set_attribute("gen_ai.request.model", model_name)
span.set_attribute("gen_ai.usage.prompt_tokens", prompt_tokens)
span.set_attribute("gen_ai.usage.completion_tokens", completion_tokens)
span.set_attribute("gen_ai.request.temperature", temperature)
span.set_attribute("gen_ai.request.top_p", top_p)
span.set_attribute("gen_ai.request.max_tokens", max_tokens)

# Optional: Langfuse-specific attributes for richer features
span.set_attribute("langfuse.observation.type", "generation")
span.set_attribute("gen_ai.usage.cost", calculated_cost)  # if you calculate it

# ⚠️ Security: DO NOT add prompt/completion content at infrastructure level!
# span.set_attribute("gen_ai.prompt.0.content", ...)  # ❌ NEVER
```

**Langfuse Will Show**:
- Cost per request
- Token distribution (prompt vs completion)
- Model usage statistics
- Latency by model/temperature/params

---

### Phase 2: Gateway Layer (Optional)

**Enhancement**: Add trace-level metadata for user/session tracking

```go
// pkg/epp/handlers/server.go - gateway.request span
span.SetAttributes(
    attribute.String("gen_ai.request.model", incomingModelName),
    attribute.String("langfuse.trace.name", "llm-inference"),
    attribute.String("langfuse.trace.user_id", userID),      // if available
    attribute.String("langfuse.trace.session_id", sessionID), // if available
    attribute.String("langfuse.trace.tags", "production,gateway"),
)
```

**Langfuse Will Show**:
- Group traces by user/session
- Track costs per user
- Filter by tags
- User-level analytics

**Security Note**: Only add user_id/session_id if they don't contain PII or you have data governance approval.

---

### Phase 3: Application Layer (Outside llm-d - Highest Value!)

If you have an application layer CALLING llm-d (RAG pipeline, agent framework, API server):

**Architecture**:
```
Your App → llm-d Gateway → vLLM
    ↓
Langfuse SDK or OTel with full GenAI attributes
```

**This is where you CAN include prompts/completions** (if security policy allows):

```python
# Your application layer (outside llm-d)
from langfuse.decorators import observe

@observe()
def rag_pipeline(user_query: str):
    # Retrieve context
    context = retriever.get_context(user_query)

    # Build prompt (Langfuse can track this!)
    prompt = f"Context: {context}\n\nQuestion: {user_query}"

    # Call llm-d (OTel trace context propagates automatically)
    response = llm_d_client.generate(prompt, model="llama-2-70b")

    return response

# Langfuse will show:
# - Full prompt (with context)
# - Full completion
# - Costs
# - Latency breakdown (retrieval + llm-d + processing)
# - User feedback integration
```

**Why Here and Not in llm-d**:
- Application layer has business context (user intent, retrieval, RAG)
- Separate security boundary (can make informed decision about logging prompts)
- Can collect user feedback on quality
- Can link to prompt templates and versions
- llm-d infrastructure should remain metadata-only for security

---

## GenAI Semantic Conventions Reference

These are the standard OTel attributes that Langfuse parses:

### Request Attributes
```
gen_ai.request.model              # "gpt-4", "llama-2-70b"
gen_ai.request.max_tokens         # integer
gen_ai.request.temperature        # float 0.0-2.0
gen_ai.request.top_p              # float 0.0-1.0
gen_ai.request.frequency_penalty  # float
gen_ai.request.presence_penalty   # float
```

### Usage Attributes
```
gen_ai.usage.prompt_tokens        # integer
gen_ai.usage.completion_tokens    # integer
gen_ai.usage.total_tokens         # integer
gen_ai.usage.cost                 # float (USD)
```

### Latency Attributes (vLLM already has these!)
```
gen_ai.latency.time_to_first_token      # seconds (TTFT)
gen_ai.latency.e2e                       # seconds (total)
gen_ai.latency.time_in_queue             # seconds
gen_ai.latency.time_in_model.prefill     # seconds
gen_ai.latency.time_in_model.decode      # seconds
```

### Content Attributes (⚠️ Use only at application layer!)
```
gen_ai.prompt.0.role           # "system", "user", "assistant"
gen_ai.prompt.0.content        # actual prompt text
gen_ai.completion.0.role       # "assistant"
gen_ai.completion.0.content    # actual completion text
```

### Langfuse-Specific Attributes (Optional)
```
langfuse.trace.name           # "rag-pipeline", "agent-task"
langfuse.trace.user_id        # user identifier
langfuse.trace.session_id     # session identifier
langfuse.trace.tags           # comma-separated tags
langfuse.observation.name     # observation name
langfuse.observation.type     # "generation", "span", "event"
langfuse.prompt.name          # link to Langfuse prompt template
langfuse.prompt.version       # integer version
```

---

## Configuration Examples

### Direct Export to Langfuse

**Gateway (Go)**:
```bash
OTEL_SERVICE_NAME=gateway-api-inference-extension
OTEL_EXPORTER_OTLP_ENDPOINT=https://cloud.langfuse.com/api/public/otel
OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic $(echo -n 'pk-lf-xxx:sk-lf-xxx' | base64)"
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1
```

**vLLM (Python)**:
```python
# In VllmConfig
vllm_config.observability_config.otlp_traces_endpoint = "https://cloud.langfuse.com/api/public/otel"

# Add headers for auth
# (implementation needed in vLLM to support OTLP headers)
```

### Dual Export (Langfuse + Jaeger) via OTel Collector

**Components → Collector**:
```bash
# Gateway and vLLM
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

**Collector → Multiple Backends**:
```yaml
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
  otlphttp/langfuse:
    endpoint: "https://cloud.langfuse.com/api/public/otel"
    headers:
      Authorization: "Basic ${LANGFUSE_AUTH}"

  jaeger:
    endpoint: "jaeger-collector:14250"
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/langfuse, jaeger]
```

---

## What You'll See in Langfuse

### Traces View
- Hierarchical trace tree (gateway → scheduler → vLLM)
- Timing waterfall (where time was spent)
- All OTel attributes as metadata
- Cost calculation automatic
- Filter by model, cost, latency, tokens

### Generations View
- Specifically for LLM generations (Langfuse auto-detects from attributes)
- Token usage charts
- Cost analysis
- Model comparison
- Quality scores (if you add user feedback)

### Dashboards
- Cost over time
- Token usage trends
- Latency percentiles (P50, P95, P99)
- Model usage distribution
- Error rates

### Prompt Management (If You Use It)
- Version prompts in Langfuse UI
- Link traces to prompt versions
- A/B test prompts
- Track prompt performance

---

## Security Considerations

### llm-d Infrastructure (Gateway + vLLM)

**DO**:
- ✅ Add `gen_ai.usage.prompt_tokens`, `gen_ai.usage.completion_tokens`
- ✅ Add `gen_ai.request.model`, `gen_ai.request.temperature`, etc.
- ✅ Add `gen_ai.latency.*` attributes
- ✅ Add `gen_ai.usage.cost` (if you calculate it)
- ✅ Add `langfuse.trace.user_id` (if it's non-PII like UUID)
- ✅ Add `langfuse.trace.tags` for filtering

**DON'T**:
- ❌ NEVER add `gen_ai.prompt.0.content` (actual prompt text)
- ❌ NEVER add `gen_ai.completion.0.content` (actual completion text)
- ❌ NEVER add any request/response body content
- ❌ NEVER add token IDs or embeddings

**Why**: llm-d is infrastructure. You don't know what prompts users are sending. Keep metadata-only.

### Application Layer (Outside llm-d)

**CAN** (if security policy allows):
- ✅ Add `gen_ai.prompt.0.content` if you own the prompts and have approval
- ✅ Add `gen_ai.completion.0.content` for quality tracking
- ✅ Use Langfuse prompt management features
- ✅ Collect user feedback

**Why**: Application layer is where business logic lives. You control the prompts and can make informed decisions about logging.

---

## Cost Considerations

### Langfuse Pricing
- **Cloud Free Tier**: 10,000 traces/month free
- **Cloud Paid**: $199/month for 500k traces, then $0.50 per 1k additional traces
- **Self-Hosted**: Open source, free to run (requires PostgreSQL + S3/compatible storage)

### Trace Volume Estimation
- 10% sampling on 1000 requests/day = 100 traces/day = 3,000 traces/month (well within free tier)
- 100% sampling on production traffic → likely need paid plan or self-hosted

### Recommendation
- Start with 10% sampling
- Use free tier or self-hosted Langfuse
- Increase sampling for specific models/users if needed
- Can use OTel Collector to sample differently per route

---

## Next Steps

1. ✅ **Continue your current setup** (OTel → Langfuse) - you're already doing it right!

2. 📈 **Phase 1**: Enhance vLLM spans with more GenAI semantic conventions
   - Add temperature, top_p, max_tokens attributes
   - Add cost calculation if not already there
   - Verify Langfuse is parsing these correctly

3. 🏷️ **Phase 2**: Add trace-level metadata in gateway (optional)
   - user_id, session_id, tags for filtering
   - Only if you have this context and it's non-PII

4. 🎯 **Phase 3**: If you have an application layer, add Langfuse instrumentation there
   - This is where prompts/completions make sense
   - Use Langfuse SDK or OTel with full GenAI attributes
   - Collect user feedback and quality metrics

5. 📊 **Monitor in Langfuse**:
   - Check cost dashboards
   - Analyze token usage trends
   - Identify expensive requests
   - Track latency by model

---

## Resources

- **Langfuse OTel Integration**: https://langfuse.com/docs/opentelemetry/get-started
- **GenAI Semantic Conventions**: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- **Langfuse Documentation**: https://langfuse.com/docs
- **OTel Collector Config**: https://opentelemetry.io/docs/collector/configuration/

---

## Questions?

**Q: Should I use Langfuse native SDK instead of OTel?**
A: No! Your current OTel approach is better because:
- Works for both Go (gateway) and Python (vLLM)
- Provides infrastructure-level tracing
- Vendor-neutral
- Can export to multiple backends

**Q: Can I send traces to both Langfuse and Jaeger?**
A: Yes! Use OpenTelemetry Collector with multiple exporters (see config above)

**Q: Should I log prompts/completions in llm-d?**
A: No at infrastructure level. Consider at application layer if you have appropriate security approval.

**Q: How does Langfuse calculate costs?**
A: From `gen_ai.usage.prompt_tokens`, `gen_ai.usage.completion_tokens`, and `gen_ai.request.model`. It has a built-in pricing database for major providers. You can also set custom pricing.

**Q: Do I need to change my OTel spans for Langfuse?**
A: No! Just add GenAI semantic convention attributes. Langfuse parses standard OTel traces.

**Q: What if I'm self-hosting Langfuse?**
A: Export to your self-hosted instance: `OTEL_EXPORTER_OTLP_ENDPOINT=https://your-langfuse.com/api/public/otel`

---

**Bottom Line**: Keep doing what you're doing (OTel → Langfuse), just add more GenAI semantic convention attributes to unlock Langfuse's full LLM observability features! 🚀
