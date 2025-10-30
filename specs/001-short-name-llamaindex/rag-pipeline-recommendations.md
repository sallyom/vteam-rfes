# RAG Pipeline Recommendations: Executive Summary

**Date**: 2025-10-30
**Context**: Reference architecture demo for Llamaindex + RamaLama RAG
**Audience**: Implementation team and technical reviewers

---

## 1. Prompt Template Design for RAG with Source Citations

### Decision
Use structured citation template with explicit constraint-based instructions.

**Recommended Template:**
```python
SYSTEM_PROMPT = """You are a helpful assistant that answers questions based solely on the provided context.

CRITICAL RULES:
1. ONLY use information from the Context section below
2. If the context doesn't contain relevant information, say "I don't have information about that in the indexed documents"
3. ALWAYS cite sources using [Source: filename, chunk X] format
4. Do not make assumptions or use external knowledge
5. Be concise but complete in your answers"""

QUERY_TEMPLATE = """Context:
{context_str}

---

Question: {query_str}

Answer (with citations):"""
```

### Rationale
- **Parseable citations**: `[Source: filename, chunk X]` format enables programmatic extraction for UI highlighting
- **Constraint-based**: "ONLY use information from context" directly addresses hallucination problem
- **Graceful degradation**: Explicit "I don't have information" instruction prevents speculation
- **Separation of concerns**: System prompt defines behavior; query template provides data
- **Human-readable**: Format is immediately understandable to users

### Alternatives Considered
- **XML-tagged citations**: `<citation source="file.pdf" chunk="42"/>` - Too rigid, harder for LLM to generate correctly
- **Numbered footnotes**: `[1]` style - Requires complex post-processing for citation resolution
- **Inline parenthetical**: `(from manual.pdf)` - Inconsistent formatting, loses chunk granularity

### Prompt Templates
```python
# Context assembly with source tracking
def format_context_for_prompt(retrieved_chunks: list[Chunk]) -> str:
    context_parts = []
    for i, chunk in enumerate(retrieved_chunks, 1):
        context_parts.append(
            f"[Chunk {i}] Source: {chunk.metadata['source_file']}, "
            f"Section: {chunk.metadata.get('section', 'N/A')}\n"
            f"{chunk.text}\n"
        )
    return "\n---\n".join(context_parts)
```

### Configuration Parameters
```yaml
prompt_config:
  system_prompt: "You are a helpful assistant..."
  max_context_length: 3000  # tokens
  citation_required: true
  citation_format: "[Source: {source_file}, chunk {chunk_id}]"
  fallback_response: "I don't have information about that in the indexed documents."
```

### Quality Considerations
- **Citation accuracy**: 100% (every statement must be cited)
- **Consistency**: Same format across all responses
- **Verifiability**: Direct mapping to database chunks for validation
- **Teachability**: Format makes RAG attribution concrete for learners

---

## 2. Context Window Management for Retrieved Documents

### Decision
Dynamic context fitting with priority-based chunk selection and token-aware assembly.

**Implementation Pattern:**
```python
class ContextWindowManager:
    def __init__(self, max_tokens: int = 3000, model_name: str = "phi-4-mini"):
        self.max_tokens = max_tokens
        self.tokenizer = self._get_tokenizer(model_name)

    def assemble_context(
        self, chunks: list[Chunk], query: str, system_prompt: str
    ) -> tuple[str, list[Chunk]]:
        # Reserve tokens for system prompt + query + response
        overhead = self._count_tokens(system_prompt) + self._count_tokens(query) + 500
        available_tokens = self.max_tokens - overhead

        # Select chunks within budget (priority: similarity score)
        selected_chunks = []
        current_tokens = 0
        for chunk in chunks:
            chunk_tokens = self._count_tokens(chunk.text) + 50
            if current_tokens + chunk_tokens <= available_tokens:
                selected_chunks.append(chunk)
                current_tokens += chunk_tokens
            else:
                break

        return self._format_context(selected_chunks), selected_chunks
```

### Rationale
- **Token-aware**: Prevents context overflow that would truncate unpredictably
- **Priority-based**: Uses similarity scores (implicit in ordering) for relevance
- **Graceful degradation**: Always includes at least top result, truncated if necessary
- **Reserve budget**: 500-token response buffer prevents cutoff mid-generation

### Alternatives Considered
- **Fixed chunk count**: Top-5 always - Ignores token limits, may overflow or waste space
- **Character-based**: Truncate to 10K chars - Characters ≠ tokens (2-4x off), arbitrary cutoff
- **Sliding window**: Multiple queries for long docs - Over-engineered for demo, high latency

### Configuration Parameters
```yaml
context_management:
  max_context_tokens: 3000
  response_reserve_tokens: 500
  min_chunks: 1
  max_chunks: 10
  truncation_strategy: "end"  # "end", "middle", or "smart_boundary"

# Model-specific defaults
models:
  phi-4-mini:
    context_window: 16384
    max_context_tokens: 3000  # Conservative
  granite-7b:
    context_window: 4096
    max_context_tokens: 2000  # Limited window
  qwen2.5:7b:
    context_window: 32768
    max_context_tokens: 6000  # Larger window
```

### Quality Considerations
- **Token counting accuracy**: Use model-specific tokenizer
- **Context quality validation**: Ensure selected chunks meet minimum similarity threshold
- **Debugging visibility**: Log token usage in `/stats` endpoint:
  ```json
  {
    "last_query": {
      "chunks_retrieved": 10,
      "chunks_included": 7,
      "context_tokens": 2847,
      "budget_tokens": 3000,
      "utilization": 0.949
    }
  }
  ```

---

## 3. Response Streaming vs. Batch Approaches

### Decision
Streaming as primary mode with buffered citations; also provide batch endpoint for comparison.

**Implementation:**
```python
async def stream_rag_response(
    query: str, chunks: list[Chunk], llm_endpoint: str
) -> AsyncIterator[StreamChunk]:
    context = format_context(chunks)
    prompt = build_prompt(query, context)
    accumulated_text = ""
    citations = extract_citation_metadata(chunks)

    async for token in llm_client.stream(prompt, endpoint=llm_endpoint):
        accumulated_text += token
        yield StreamChunk(
            delta=token,
            accumulated_text=accumulated_text,
            citations=citations,  # Include in every chunk
            done=False
        )

    yield StreamChunk(
        delta="", accumulated_text=accumulated_text,
        citations=citations, done=True
    )

# FastAPI endpoint
@app.post("/query/stream")
async def query_stream(query: QueryRequest):
    chunks = await vector_store.similarity_search(query.text)
    async def generate():
        async for chunk in stream_rag_response(query.text, chunks, llm_endpoint):
            yield f"data: {chunk.model_dump_json()}\n\n"
        yield "data: [DONE]\n\n"
    return StreamingResponse(generate(), media_type="text/event-stream")
```

### Rationale
- **Perceived performance**: Immediate visual feedback makes 2-3s generation feel faster
- **Progressive disclosure**: Users see answer forming, building trust
- **Citation availability**: Full citation list in every chunk enables immediate UI display
- **Production pattern**: Matches modern LLM app standards (ChatGPT, Claude)
- **Debugging visibility**: Token-by-token generation makes LLM behavior observable

### Alternatives Considered
- **Batch-only**: Simpler but longer perceived latency, doesn't match modern UX expectations
- **Stream without citations**: Poor UX (sources only at end)
- **WebSocket**: Over-engineered for demo, SSE is simpler and curl-testable

### Configuration Parameters
```yaml
streaming:
  enabled: true
  chunk_size: 1  # tokens per chunk (1=real-time, higher=buffered)
  include_citations_in_stream: true
  timeout_seconds: 30

batch:
  enabled: true  # Also provide for comparison
  max_generation_tokens: 500
  timeout_seconds: 10
```

### Quality Considerations
- **Error handling in streams**: Send errors as final stream chunk with `error` field
- **Client reconnection**: Document retry logic for clients
- **Testing**: Verify streaming characteristics (multiple chunks, done flag, citations present)

---

## 4. Handling Queries About Non-Indexed Content

### Decision
Similarity threshold with explicit "not found" responses.

**Implementation:**
```python
class RAGQueryHandler:
    def __init__(self, similarity_threshold: float = 0.7):
        self.similarity_threshold = similarity_threshold

    async def query(self, query_text: str, top_k: int = 5) -> RAGResponse:
        chunks = await self.vector_store.similarity_search(query_text, k=top_k)

        # Filter by similarity threshold
        relevant_chunks = [
            c for c in chunks
            if c.similarity_score >= self.similarity_threshold
        ]

        if not relevant_chunks:
            return RAGResponse(
                answer=(
                    f"I don't have relevant information about that in the indexed documents. "
                    f"The most similar content I found had a relevance score of "
                    f"{chunks[0].similarity_score:.2f} (threshold: {self.similarity_threshold}), "
                    "which is below the confidence threshold for a reliable answer."
                ),
                sources=[],
                confidence="none",
                metadata={"reason": "no_relevant_chunks", "top_similarity": chunks[0].similarity_score}
            )

        response = await self.llm_generate(query_text, relevant_chunks)
        return RAGResponse(
            answer=response.text,
            sources=relevant_chunks,
            confidence="high" if relevant_chunks[0].similarity_score > 0.85 else "medium"
        )
```

### Rationale
- **Explicit threshold**: 0.7 cutoff prevents hallucination from marginal chunks
- **Transparent failure**: Makes system boundaries visible (critical for learning)
- **Educational value**: Including similarity scores teaches relevance decisions
- **Production pattern**: Models real-world RAG handling of out-of-domain queries
- **Prevents hallucination**: Never asks LLM to generate from low-relevance context

### Alternatives Considered
- **Always generate**: No threshold - High hallucination risk, unprofessional
- **Ask LLM to decide**: Include "say if no info" in prompt - Unreliable, LLM may generate anyway
- **Hybrid approach**: Different thresholds for different confidence levels - Good for production, too complex for demo

### Configuration Parameters
```yaml
retrieval:
  similarity_threshold: 0.7

  # Threshold tiers for confidence reporting
  thresholds:
    high_confidence: 0.85
    medium_confidence: 0.7
    low_confidence: 0.5  # Still filtered out

  fallback:
    enabled: true
    message_template: "I don't have relevant information about {query}..."
    include_diagnostic_info: true  # Show similarity scores
```

### Quality Considerations
- **Threshold tuning guidance**:
  - 0.9-1.0: Near-exact match (rare, very high precision)
  - 0.8-0.9: Strong match (high precision, good recall)
  - 0.7-0.8: Moderate match (balanced, recommended default)
  - 0.6-0.7: Weak match (high recall, lower precision)
  - <0.6: Poor match (high hallucination risk)
- **Observability**: Log relevance distribution for tuning analysis
- **Testing**: Parametrize tests with queries of varying relevance

---

## 5. Retrieval Parameter Tuning (top-k, similarity threshold)

### Decision
Configurable defaults with runtime override capability.

**Configuration:**
```python
class RetrievalConfig(BaseModel):
    top_k: int = Field(default=5, ge=1, le=20)
    similarity_threshold: float = Field(default=0.7, ge=0.0, le=1.0)
    diversity_penalty: float = Field(default=0.0, ge=0.0, le=1.0)
    rerank_enabled: bool = Field(default=False)

# API endpoint with override
@app.post("/query")
async def query(query: QueryRequest, retrieval_override: Optional[RetrievalConfig] = None):
    config = retrieval_override or default_retrieval_config
    chunks = await retrieve(query.text, config)
    return await generate_response(query.text, chunks)
```

### Rationale
- **Sensible defaults**: top_k=5, threshold=0.7 work well out-of-the-box
- **Runtime flexibility**: Enables experimentation without redeployment (critical for demo)
- **Bounded ranges**: Validation prevents pathological values
- **Progressive disclosure**: Basic users use defaults, advanced users experiment
- **Observable impact**: Parameter changes have immediate, visible effects

### Alternatives Considered
- **Fixed parameters**: Config file only - Too rigid for demo/learning
- **Unlimited range**: No bounds - Easy to misconfigure and break system
- **Automatic tuning**: System auto-adjusts - Hides RAG mechanics, defeats learning purpose

### Configuration Parameters
```yaml
retrieval:
  defaults:
    top_k: 5
    similarity_threshold: 0.7
    diversity_penalty: 0.0
    rerank_enabled: false

  limits:
    max_top_k: 20
    min_top_k: 1
    max_threshold: 1.0
    min_threshold: 0.0
```

**Parameter Guidelines:**

| Use Case | top_k | threshold | Reasoning |
|----------|-------|-----------|-----------|
| Precise technical queries | 3-5 | 0.75-0.80 | Focused, high-relevance |
| Broad exploratory queries | 7-10 | 0.70-0.75 | More coverage |
| Multi-aspect questions | 8-12 | 0.70 | Need diverse chunks |
| Summary/overview requests | 10-15 | 0.65-0.70 | Comprehensive view |

### Quality Considerations
- **Parameter interaction warnings**:
  ```python
  # High threshold + low top_k may return nothing
  if threshold > 0.85 and top_k < 5:
      warn("Consider increasing top_k or lowering threshold")

  # Low threshold + high top_k wastes context
  if threshold < 0.6 and top_k > 10:
      warn("Consider raising threshold or reducing top_k")
  ```
- **Telemetry**: Expose parameter usage stats in `/stats` endpoint
- **Documentation**: Provide experimentation guide with recommended combinations

---

## 6. Error Handling Patterns

### Decision
Layered error handling with user-friendly messages and actionable suggestions.

**Implementation:**
```python
class RAGException(Exception):
    def __init__(self, message: str, user_message: str, details: dict = None):
        self.message = message  # Technical details for logs
        self.user_message = user_message  # User-facing message
        self.details = details or {}

class DatabaseConnectionError(RAGException): pass
class EmbeddingModelMismatchError(RAGException): pass
class LLMGenerationError(RAGException): pass
class ContextWindowExceededError(RAGException): pass

@app.exception_handler(RAGException)
async def rag_exception_handler(request: Request, exc: RAGException):
    logger.error("RAG operation failed", error=exc.message, details=exc.details)

    status_map = {
        DatabaseConnectionError: 503,
        EmbeddingModelMismatchError: 500,
        LLMGenerationError: 502,
        ContextWindowExceededError: 400,
    }
    status_code = status_map.get(type(exc), 500)

    return JSONResponse(
        status_code=status_code,
        content={
            "error": exc.user_message,
            "details": exc.details if app.debug_mode else {},
            "suggestion": get_error_suggestion(type(exc))
        }
    )
```

### Rationale
- **User-friendly messages**: Separate technical details from user-facing messages
- **Structured error types**: Enable precise handling and status code mapping
- **Actionable suggestions**: Each error type includes specific remediation steps
- **Logging integration**: Structured context enables post-mortem analysis
- **Progressive detail**: Debug mode exposes technical details; production hides them

### Alternatives Considered
- **Generic exceptions**: Simple but loses specificity, hard to provide targeted guidance
- **HTTP-only errors**: Return only status codes - Loses context, poor UX
- **Overly detailed**: Full stack traces to users - Security risk, overwhelming

### Configuration Parameters
```yaml
error_handling:
  debug_mode: true  # Include technical details in responses
  log_level: "INFO"
  include_suggestions: true

  timeouts:
    database_query: 5  # seconds
    llm_generation: 30
    total_request: 35

  retries:
    database_operations: 3
    llm_generation: 1  # Usually not worth retrying
    exponential_backoff: true
```

**Error Suggestion Mapping:**
```python
suggestions = {
    DatabaseConnectionError: "Check that PostgreSQL database is running and accessible.",
    EmbeddingModelMismatchError: "Verify embedding model in config matches model used to process documents.",
    LLMGenerationError: "Check that LLM server (RamaLama/Ollama) is running and model is loaded.",
    ContextWindowExceededError: "Try a more specific query or reduce the top_k parameter.",
}
```

### Quality Considerations
- **Health check integration**: `/health` endpoint validates all components
  ```json
  {
    "status": "healthy",
    "checks": {
      "database": {"status": "ok"},
      "llm": {"status": "ok"},
      "embedding_model": {"status": "ok"}
    }
  }
  ```
- **Retry patterns**: Exponential backoff for transient failures
- **Error recovery**: Test all error paths for graceful degradation

---

## Overall Recommendations

### Recommended Pipeline Configuration

```yaml
# Complete configuration for demo
pipeline:
  prompts:
    system_prompt: "You are a helpful assistant..." # Structured citation template
    citation_format: "[Source: {source_file}, chunk {chunk_id}]"

  context:
    max_tokens: 3000
    response_reserve: 500
    truncation_strategy: "priority_based"

  retrieval:
    top_k: 5
    similarity_threshold: 0.7
    runtime_override: true

  response:
    streaming: true  # Primary mode
    batch: true  # Also provide for comparison
    include_citations_in_stream: true

  errors:
    debug_mode: true  # For demo
    user_friendly_messages: true
    include_suggestions: true
```

### Implementation Priority

**Phase 1: Core Functionality** (MVP)
1. Structured prompt template with citations
2. Token-aware context assembly
3. Similarity threshold filtering
4. Batch query endpoint
5. Basic error handling

**Phase 2: Production Quality**
6. Streaming response endpoint
7. Layered error handling with user messages
8. Health check endpoint
9. Runtime parameter override
10. Comprehensive logging

**Phase 3: Demo Polish**
11. `/stats` endpoint for observability
12. Parameter tuning documentation
13. Example queries with expected behavior
14. Troubleshooting guide

### Key Quality Metrics

**Response Quality:**
- Citation accuracy: 100% (every statement cited)
- Relevance: Avg similarity score >0.75 for included chunks
- Out-of-scope handling: 0% hallucination rate

**Performance:**
- Query latency: <5s end-to-end
- Context assembly: <500ms
- Streaming first-token: <1s

**Reliability:**
- Startup success rate: >95%
- Query success rate: >99% (excluding user errors)
- Graceful degradation: All error paths tested

### What This Teaches Learners

1. **Citation discipline**: RAG systems must attribute every claim
2. **Token budgeting**: Context windows are finite, require management
3. **Threshold tuning**: Similarity scores determine quality-relevance tradeoff
4. **Graceful failure**: Systems must explicitly handle out-of-scope queries
5. **Streaming UX**: Modern LLM apps stream for perceived performance
6. **Observable behavior**: Instrumentation enables understanding and tuning

---

## Next Steps

1. Review these recommendations with implementation team
2. Finalize configuration defaults for target corpus
3. Design API contracts based on these patterns
4. Create test scenarios covering all quality considerations
5. Document parameter tuning guide for demo users

**Questions for Review:**
- Confirm default thresholds (0.7, top_k=5) are appropriate
- Decide: streaming-only or include batch endpoint?
- Set debug_mode default (true for demo accessibility?)

---

**Full research details**: See `/workspace/sessions/agentic-session-1761836300/workspace/vteam-rfes/specs/001-short-name-llamaindex/research-rag-pipeline-best-practices.md`
