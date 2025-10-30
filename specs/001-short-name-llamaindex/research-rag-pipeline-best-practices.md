# RAG Pipeline Best Practices: Llamaindex + Local LLM Research

**Date**: 2025-10-30
**Purpose**: Technical research for Llamaindex + RamaLama RAG demo
**Scope**: Best practices for production-quality RAG pipelines suitable for demo/learning contexts

---

## Executive Summary

This research evaluates best practices for building RAG pipelines with Llamaindex orchestration and local LLM serving (RamaLama/Ollama). The goal is to establish patterns that demonstrate production-quality approaches while remaining accessible to learners.

**Key Finding**: A prompt-template-driven approach with streaming responses, explicit citation formatting, and graceful handling of out-of-scope queries provides the best balance of quality and teachability for a reference demo.

---

## 1. Prompt Template Design for RAG with Source Citations

### Decision: Structured Citation Template with Separation of Concerns

**Recommended Template Pattern:**

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

1. **Explicit Citation Format**: The `[Source: filename, chunk X]` format is:
   - **Parseable**: Can be extracted programmatically for UI highlighting
   - **Human-readable**: Users immediately understand source attribution
   - **Verifiable**: Direct mapping to database chunks for validation

2. **Constraint-Based Instruction**: "ONLY use information from the provided context" directly addresses the hallucination problem common in RAG systems. This is critical for demos where users need to trust outputs.

3. **Graceful Degradation**: The "I don't have information" instruction prevents the LLM from speculating when context is insufficient, making quality issues visible to learners.

4. **Separation of Concerns**: System prompt defines behavior rules; query template provides runtime data. This separation makes it easy to tune behavior without changing query construction code.

### Alternatives Considered

**Alternative 1: XML-Tagged Citations**
```xml
<answer>
  The system requirements are...
  <citation source="manual.pdf" chunk="42"/>
</answer>
```
- **Pros**: Strict structure, easier parsing
- **Cons**: More verbose, harder for LLM to generate correctly, less human-friendly
- **Verdict**: Too rigid for demo context; better for production systems with guaranteed parsing

**Alternative 2: Numbered Footnote Style**
```
The system requirements are X, Y, Z [1].

[1] installation-guide.pdf, page 5
```
- **Pros**: Academic familiarity, clean reading flow
- **Cons**: Requires post-processing to resolve numbers; citation management complexity
- **Verdict**: More complexity than value for demo use case

**Alternative 3: Inline Parenthetical**
```
The system requires 4GB RAM (from manual.pdf) and 20GB storage (from specs.pdf).
```
- **Pros**: Very natural reading flow
- **Cons**: Inconsistent formatting, hard to parse, loses chunk granularity
- **Verdict**: Too informal for production reference architecture

### Prompt Template Components

**Context Assembly Pattern:**
```python
def format_context_for_prompt(retrieved_chunks: list[Chunk]) -> str:
    """Format retrieved chunks into context string with source tracking"""
    context_parts = []
    for i, chunk in enumerate(retrieved_chunks, 1):
        context_parts.append(
            f"[Chunk {i}] Source: {chunk.metadata['source_file']}, "
            f"Section: {chunk.metadata.get('section', 'N/A')}\n"
            f"{chunk.text}\n"
        )
    return "\n---\n".join(context_parts)
```

**Why This Works:**
- Each chunk is labeled with an ID the LLM can reference
- Source metadata is visible in context, enabling citation
- Separator (`---`) provides clear chunk boundaries for LLM
- Metadata includes section for finer-grained attribution

### Configuration Parameters

```yaml
prompt_config:
  system_prompt: "You are a helpful assistant..."  # Full prompt from above
  max_context_length: 3000  # tokens, adjusted for model context window
  citation_required: true
  citation_format: "[Source: {source_file}, chunk {chunk_id}]"
  fallback_response: "I don't have information about that in the indexed documents."
```

---

## 2. Context Window Management for Retrieved Documents

### Decision: Dynamic Context Fitting with Priority-Based Chunk Selection

**Recommended Approach:**

```python
class ContextWindowManager:
    """Manages context assembly within token budget"""

    def __init__(self, max_tokens: int = 3000, model_name: str = "phi-4-mini"):
        self.max_tokens = max_tokens
        self.tokenizer = self._get_tokenizer(model_name)

    def assemble_context(
        self,
        chunks: list[Chunk],
        query: str,
        system_prompt: str
    ) -> tuple[str, list[Chunk]]:
        """
        Assemble context within token budget using priority selection

        Returns:
            - Formatted context string
            - List of chunks actually included (for citation tracking)
        """
        # Reserve tokens for system prompt + query + response
        overhead_tokens = (
            self._count_tokens(system_prompt) +
            self._count_tokens(query) +
            500  # Reserve for response generation
        )
        available_tokens = self.max_tokens - overhead_tokens

        # Priority: similarity score (chunks already sorted by retrieval)
        selected_chunks = []
        current_tokens = 0

        for chunk in chunks:
            chunk_tokens = self._count_tokens(chunk.text) + 50  # metadata overhead
            if current_tokens + chunk_tokens <= available_tokens:
                selected_chunks.append(chunk)
                current_tokens += chunk_tokens
            else:
                break  # Stop when budget exhausted

        if not selected_chunks:
            # Fallback: include at least the top result, truncated
            selected_chunks = [self._truncate_chunk(chunks[0], available_tokens)]

        return self._format_context(selected_chunks), selected_chunks
```

### Rationale

1. **Token-Aware Assembly**: Counting tokens (not characters) prevents context overflow that would truncate unpredictably. This is critical for local models with limited context windows (e.g., Phi-4-mini: 16K tokens).

2. **Priority-Based Selection**: Using similarity scores (implicit in chunk ordering) ensures highest-quality context. This trades off completeness for relevance, which is appropriate for demo scenarios.

3. **Graceful Degradation**: If no chunks fit, truncate the top result rather than failing. This ensures the system always produces an answer attempt, making failure modes visible to learners.

4. **Reserve Budget**: Allocating 500 tokens for response generation prevents cutoff mid-generation. This is learned from production RAG systems where insufficient headroom causes incomplete answers.

### Configuration Parameters

```yaml
context_management:
  max_context_tokens: 3000
  response_reserve_tokens: 500
  min_chunks: 1  # Always include at least top result
  max_chunks: 10  # Upper bound even if tokens available
  truncation_strategy: "end"  # "end", "middle", or "smart_boundary"
```

**Model-Specific Defaults:**

| Model | Context Window | Max Context Tokens | Reserve | Reasoning |
|-------|----------------|--------------------|---------|-----------|
| phi-4-mini | 16K | 3000 | 500 | Conservative for stability |
| granite-7b | 4K | 2000 | 500 | Limited window, tighter budget |
| qwen2.5:7b | 32K | 6000 | 500 | Larger window, more context |

### Alternatives Considered

**Alternative 1: Fixed Chunk Count**
```python
# Always use top-5 chunks regardless of length
context_chunks = chunks[:5]
```
- **Pros**: Simple, predictable
- **Cons**: Ignores token limits, may overflow context or waste space
- **Verdict**: Too naive for production reference; hides important token management concern

**Alternative 2: Character-Based Truncation**
```python
# Truncate to 10K characters
context = "\n".join(chunk.text for chunk in chunks)[:10000]
```
- **Pros**: Very simple
- **Cons**: Characters ≠ tokens (can be 2-4x off); arbitrary cutoff may split mid-sentence
- **Verdict**: Inaccurate and unprofessional for demo quality

**Alternative 3: Sliding Window with Overlap**
```python
# If context exceeds budget, use sliding window across multiple queries
```
- **Pros**: Comprehensive coverage for long documents
- **Cons**: Multiple LLM calls per query; latency unacceptable for demo; complex merging logic
- **Verdict**: Over-engineered for demo use case; appropriate for production only

### Quality Considerations

**Token Counting Accuracy:**
- Use the same tokenizer as the target model (critical for accuracy)
- For mixed models (Ollama serving), use conservative estimates (1.5 chars/token)
- Log actual token counts for tuning: `logger.info(f"Context tokens: {actual}, budget: {max}")`

**Context Quality Validation:**
```python
def validate_context_quality(chunks: list[Chunk], min_similarity: float = 0.7) -> bool:
    """Ensure selected chunks meet quality threshold"""
    if not chunks:
        return False
    return all(chunk.similarity_score >= min_similarity for chunk in chunks)
```

**Debugging Aid:**
```python
# Include in /stats endpoint for demo transparency
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

### Decision: Streaming with Buffered Citations

**Recommended Approach:**

```python
async def stream_rag_response(
    query: str,
    chunks: list[Chunk],
    llm_endpoint: str
) -> AsyncIterator[StreamChunk]:
    """
    Stream LLM response with accumulated citations

    Yields:
        StreamChunk with partial_response and citations_so_far
    """
    context = format_context(chunks)
    prompt = build_prompt(query, context)

    accumulated_text = ""
    citations = extract_citation_metadata(chunks)

    async for token in llm_client.stream(prompt, endpoint=llm_endpoint):
        accumulated_text += token

        # Yield incremental update
        yield StreamChunk(
            delta=token,
            accumulated_text=accumulated_text,
            citations=citations,  # Always include full citation list
            done=False
        )

    # Final chunk with complete response
    yield StreamChunk(
        delta="",
        accumulated_text=accumulated_text,
        citations=citations,
        done=True
    )
```

**FastAPI Endpoint:**
```python
@app.post("/query/stream")
async def query_stream(query: QueryRequest):
    """Streaming RAG query endpoint"""

    # Retrieve relevant chunks
    chunks = await vector_store.similarity_search(
        query.text,
        k=query.top_k,
        threshold=query.threshold
    )

    # Stream response
    async def generate():
        async for chunk in stream_rag_response(query.text, chunks, llm_endpoint):
            yield f"data: {chunk.model_dump_json()}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

### Rationale

1. **Perceived Performance**: Streaming provides immediate visual feedback, making 2-3 second generation feel faster than batch. This is critical for demo engagement.

2. **Progressive Disclosure**: Users see the answer forming, which builds trust. If the answer goes off-track, users can interrupt (though demo doesn't implement interrupt).

3. **Citation Availability**: Including full citation list in every chunk ensures UI can display sources immediately, not just at the end.

4. **Production Pattern**: Streaming is the modern standard for LLM apps (ChatGPT, Claude, etc.). Demo should reflect current best practices.

5. **Debugging Visibility**: Streaming exposes token-by-token generation, making LLM behavior more observable for learners.

### Alternatives Considered

**Alternative 1: Batch-Only Response**
```python
@app.post("/query")
async def query_batch(query: QueryRequest):
    chunks = await retrieve(query.text)
    response = await llm_generate(chunks)  # Wait for full response
    return {"response": response, "sources": chunks}
```
- **Pros**: Simpler implementation, easier error handling
- **Cons**: Longer perceived latency, doesn't match modern UX expectations
- **Verdict**: Acceptable for demo but not preferred; include as alternative endpoint

**Alternative 2: Streaming Without Citations**
```python
# Stream tokens only, send citations at end
async for token in llm_stream():
    yield {"delta": token}
yield {"citations": citations}  # Final message
```
- **Pros**: Minimal per-chunk payload
- **Cons**: UI can't show sources until end; poor UX for slow generation
- **Verdict**: Worse UX than buffered approach; no implementation benefit

**Alternative 3: WebSocket-Based Streaming**
```python
@app.websocket("/query/ws")
async def query_websocket(websocket: WebSocket):
    await websocket.accept()
    # Bidirectional streaming
```
- **Pros**: True bidirectional communication, can send cancel signals
- **Cons**: More complex client/server code, harder to demonstrate with curl
- **Verdict**: Over-engineered for demo; SSE is simpler and sufficient

### Configuration Parameters

```yaml
streaming:
  enabled: true
  chunk_size: 1  # tokens per chunk (1 = real-time, higher = buffered)
  include_citations_in_stream: true
  timeout_seconds: 30

# Also provide batch endpoint
batch:
  enabled: true
  max_generation_tokens: 500
  timeout_seconds: 10
```

### Quality Considerations

**Error Handling in Streams:**
```python
async def stream_rag_response(...):
    try:
        async for token in llm_client.stream(prompt):
            yield StreamChunk(delta=token, ...)
    except LLMException as e:
        # Send error as final stream chunk
        yield StreamChunk(
            delta="",
            accumulated_text=accumulated_text,
            citations=citations,
            done=True,
            error=str(e)
        )
```

**Client-Side Reconnection:**
```javascript
// Example JS client with automatic reconnect
const eventSource = new EventSource('/query/stream');
eventSource.onerror = () => {
    setTimeout(() => reconnect(), 1000);  // Retry after 1s
};
```

**Testing Strategy:**
```python
# Test streaming behavior
@pytest.mark.asyncio
async def test_streaming_response():
    chunks = []
    async for chunk in stream_rag_response(query, context):
        chunks.append(chunk)

    # Verify streaming characteristics
    assert len(chunks) > 1, "Should yield multiple chunks"
    assert chunks[-1].done is True, "Last chunk should be marked done"
    assert all(c.citations for c in chunks), "All chunks should include citations"
```

---

## 4. Handling Queries About Non-Indexed Content

### Decision: Similarity Threshold with Explicit "Not Found" Responses

**Recommended Approach:**

```python
class RAGQueryHandler:
    """Handles RAG queries with fallback for low-relevance scenarios"""

    def __init__(self, similarity_threshold: float = 0.7):
        self.similarity_threshold = similarity_threshold

    async def query(self, query_text: str, top_k: int = 5) -> RAGResponse:
        """
        Execute RAG query with fallback for insufficient context
        """
        # Retrieve chunks
        chunks = await self.vector_store.similarity_search(
            query_text,
            k=top_k
        )

        # Filter by similarity threshold
        relevant_chunks = [
            c for c in chunks
            if c.similarity_score >= self.similarity_threshold
        ]

        # Decision point: sufficient context?
        if not relevant_chunks:
            return self._no_relevant_content_response(query_text, chunks)

        # Proceed with generation
        response = await self.llm_generate(query_text, relevant_chunks)

        return RAGResponse(
            answer=response.text,
            sources=relevant_chunks,
            confidence="high" if relevant_chunks[0].similarity_score > 0.85 else "medium"
        )

    def _no_relevant_content_response(
        self,
        query: str,
        chunks: list[Chunk]
    ) -> RAGResponse:
        """Generate helpful response when no relevant content found"""

        # Option 1: Return explicit message (preferred for demo)
        return RAGResponse(
            answer=(
                "I don't have relevant information about that in the indexed documents. "
                f"The most similar content I found had a relevance score of "
                f"{chunks[0].similarity_score:.2f} (threshold: {self.similarity_threshold}), "
                "which is below the confidence threshold for a reliable answer."
            ),
            sources=[],
            confidence="none",
            metadata={
                "reason": "no_relevant_chunks",
                "top_similarity": chunks[0].similarity_score if chunks else 0.0,
                "threshold": self.similarity_threshold
            }
        )
```

### Rationale

1. **Explicit Threshold**: Using 0.7 similarity as a hard cutoff prevents hallucination from marginally-relevant chunks. This is based on empirical observation: scores below 0.7 often indicate semantic mismatch.

2. **Transparent Failure**: Returning explicit "not found" messages makes system boundaries visible, which is critical for learning. Users understand the system's limitations.

3. **Educational Value**: Including the actual similarity score in the response teaches users how RAG systems make relevance decisions.

4. **Production Pattern**: Production RAG systems must gracefully handle out-of-domain queries. Demo should model this behavior.

5. **Prevents Hallucination**: Never asking the LLM to generate from low-relevance context eliminates the most common RAG failure mode.

### Alternatives Considered

**Alternative 1: Always Generate (No Threshold)**
```python
# Generate from top-k results regardless of similarity
response = await llm_generate(query, chunks)
```
- **Pros**: Simpler logic, always provides an answer
- **Cons**: High hallucination risk, misleads users about system capabilities
- **Verdict**: Unprofessional for demo; hides critical quality concern

**Alternative 2: Ask LLM to Decide**
```python
# Include in prompt: "If the context doesn't contain relevant information, say so"
response = await llm_generate_with_instruction(query, chunks)
```
- **Pros**: LLM can assess relevance semantically
- **Cons**: Unreliable (LLM may generate anyway), costs extra tokens, slower
- **Verdict**: Not trustworthy enough for demo; use as secondary check only

**Alternative 3: Hybrid Approach**
```python
if max_similarity < 0.7:
    return no_content_response()
elif max_similarity < 0.85:
    # Let LLM decide with explicit instruction
    response = await llm_generate_with_caution(query, chunks)
else:
    # High confidence, generate normally
    response = await llm_generate(query, chunks)
```
- **Pros**: Balances safety and utility
- **Cons**: More complex logic, harder to explain in demo
- **Verdict**: Good for production, but threshold-only is clearer for learning

### Configuration Parameters

```yaml
retrieval:
  similarity_threshold: 0.7  # Minimum score for relevance
  top_k: 5

  # Threshold tiers for confidence reporting
  thresholds:
    high_confidence: 0.85  # "Highly relevant"
    medium_confidence: 0.7  # "Relevant"
    low_confidence: 0.5     # "Possibly relevant" (still filtered out)

  # Fallback behavior
  fallback:
    enabled: true
    message_template: "I don't have relevant information about {query}..."
    include_diagnostic_info: true  # Show similarity scores in response
```

### Quality Considerations

**Threshold Tuning Guidance:**
```python
# Include in demo documentation
SIMILARITY_THRESHOLD_GUIDE = """
Similarity Threshold Tuning:

0.9-1.0: Near-exact semantic match (rare, very high precision)
0.8-0.9: Strong semantic match (high precision, good recall)
0.7-0.8: Moderate semantic match (balanced, recommended default)
0.6-0.7: Weak semantic match (high recall, lower precision)
<0.6:   Poor semantic match (high hallucination risk)

Recommended: Start at 0.7, adjust based on your corpus:
- Technical docs with precise terminology: 0.75-0.8
- General knowledge with varied phrasing: 0.65-0.7
- Multi-domain corpus: 0.7 (balanced)
"""
```

**Observability:**
```python
# Log for tuning analysis
logger.info(
    "Query relevance distribution",
    query=query_text,
    top_score=chunks[0].similarity_score,
    threshold=self.similarity_threshold,
    relevant_chunks=len(relevant_chunks),
    total_chunks=len(chunks)
)
```

**Testing Edge Cases:**
```python
@pytest.mark.parametrize("query,expected_behavior", [
    ("random gibberish xyzabc", "no_relevant_content"),
    ("system requirements", "high_confidence"),
    ("vaguely related topic", "medium_confidence"),
])
async def test_query_relevance_handling(query, expected_behavior):
    response = await rag_handler.query(query)
    assert response.confidence == expected_behavior
```

---

## 5. Retrieval Parameter Tuning (top-k, similarity threshold)

### Decision: Configurable Defaults with Runtime Override

**Recommended Configuration:**

```python
class RetrievalConfig(BaseModel):
    """Retrieval parameters with validated defaults"""

    # Core parameters
    top_k: int = Field(
        default=5,
        ge=1,
        le=20,
        description="Number of chunks to retrieve"
    )
    similarity_threshold: float = Field(
        default=0.7,
        ge=0.0,
        le=1.0,
        description="Minimum similarity score for relevance"
    )

    # Advanced parameters
    diversity_penalty: float = Field(
        default=0.0,
        ge=0.0,
        le=1.0,
        description="MMR diversity penalty (0=pure similarity, 1=max diversity)"
    )
    rerank_enabled: bool = Field(
        default=False,
        description="Apply cross-encoder reranking after retrieval"
    )

    @property
    def effective_k(self) -> int:
        """Fetch size accounting for threshold filtering"""
        # Fetch 2x to account for threshold filtering
        return min(self.top_k * 2, 20)

# API endpoint with parameter override
@app.post("/query")
async def query(
    query: QueryRequest,
    retrieval_override: Optional[RetrievalConfig] = None
):
    """
    Query with optional parameter override

    Example:
        POST /query
        {
            "text": "system requirements",
            "retrieval": {
                "top_k": 10,
                "similarity_threshold": 0.75
            }
        }
    """
    config = retrieval_override or default_retrieval_config
    chunks = await retrieve(query.text, config)
    return await generate_response(query.text, chunks)
```

### Rationale

1. **Sensible Defaults**: `top_k=5` and `threshold=0.7` work well for most use cases based on empirical testing. Demo should work out-of-the-box.

2. **Runtime Flexibility**: Allowing override via API enables experimentation without redeployment. This is critical for demo context where users explore behavior.

3. **Bounded Ranges**: Validation prevents pathological values (e.g., `top_k=1000` would overwhelm context window, `threshold=0.1` would accept noise).

4. **Progressive Disclosure**: Basic users use defaults; advanced users experiment with parameters. This makes demo accessible to both beginners and experts.

5. **Observable Impact**: Parameter changes have immediate, visible effects on results, making RAG behavior concrete for learners.

### Parameter Guidelines

**top_k Selection:**

| Use Case | Recommended top_k | Reasoning |
|----------|-------------------|-----------|
| Precise technical queries | 3-5 | Focused, high-relevance context |
| Broad exploratory queries | 7-10 | More coverage, accept lower avg relevance |
| Multi-aspect questions | 8-12 | Need diverse chunks to cover all aspects |
| Summary/overview requests | 10-15 | Comprehensive view of corpus |

**similarity_threshold Selection:**

| Corpus Type | Recommended Threshold | Reasoning |
|-------------|----------------------|-----------|
| API documentation | 0.75-0.80 | Precise terminology, low ambiguity |
| User manuals | 0.70-0.75 | Mix of technical and conversational |
| Marketing content | 0.65-0.70 | Varied phrasing, more semantic variance |
| Mixed corpus | 0.70 (default) | Balanced for diverse content |

### Alternatives Considered

**Alternative 1: Fixed Parameters Only**
```yaml
# No runtime override, config file only
retrieval:
  top_k: 5
  similarity_threshold: 0.7
```
- **Pros**: Simpler API, consistent behavior
- **Cons**: Can't experiment without redeployment; poor demo UX
- **Verdict**: Too rigid for demo/learning context

**Alternative 2: Unlimited Parameter Range**
```python
top_k: int = Field(description="Any positive integer")
```
- **Pros**: Maximum flexibility
- **Cons**: Easy to misconfigure (e.g., fetch 10,000 chunks), no guidance
- **Verdict**: Dangerous for demo; users can break system easily

**Alternative 3: Automatic Tuning**
```python
# System auto-adjusts parameters based on query characteristics
top_k = auto_determine_k(query)
threshold = auto_determine_threshold(query)
```
- **Pros**: Optimal results without user expertise
- **Cons**: Hides RAG mechanics, defeats learning purpose, complex implementation
- **Verdict**: Inappropriate for demo; obscures concepts being taught

### Configuration Parameters

```yaml
# config.yaml
retrieval:
  defaults:
    top_k: 5
    similarity_threshold: 0.7
    diversity_penalty: 0.0
    rerank_enabled: false

  # Validation bounds
  limits:
    max_top_k: 20
    min_top_k: 1
    max_threshold: 1.0
    min_threshold: 0.0

  # Advanced: Adaptive tuning (non-MVP)
  adaptive:
    enabled: false
    adjust_k_by_query_length: true  # Longer queries → higher k
    adjust_threshold_by_corpus_size: true  # Larger corpus → higher threshold
```

### Quality Considerations

**Parameter Interaction Effects:**
```python
def validate_parameter_consistency(config: RetrievalConfig) -> list[str]:
    """Check for problematic parameter combinations"""
    warnings = []

    # High threshold + low top_k may return nothing
    if config.similarity_threshold > 0.85 and config.top_k < 5:
        warnings.append(
            "High threshold (>0.85) with low top_k (<5) may return no results. "
            "Consider increasing top_k or lowering threshold."
        )

    # Low threshold + high top_k wastes context
    if config.similarity_threshold < 0.6 and config.top_k > 10:
        warnings.append(
            "Low threshold (<0.6) with high top_k (>10) may include irrelevant chunks. "
            "Consider raising threshold or reducing top_k."
        )

    return warnings
```

**Documentation for Demo Users:**
```markdown
## Tuning Retrieval Parameters

### Quick Start
Leave defaults unchanged for most use cases.

### Experimentation
Try these parameter combinations to see their effects:

1. **High Precision**: `top_k=3, threshold=0.8`
   - Returns only highly confident results
   - Risk: May miss relevant but differently-phrased content

2. **High Recall**: `top_k=10, threshold=0.65`
   - Returns broader set of potentially relevant content
   - Risk: May include marginally relevant chunks

3. **Balanced** (default): `top_k=5, threshold=0.7`
   - Good balance for most queries

### Observing Effects
Use `/stats` endpoint to see:
- Distribution of similarity scores
- Average chunks returned per query
- Queries with no relevant results (threshold filtering)
```

**Telemetry for Tuning:**
```python
# Expose in /stats endpoint
{
  "retrieval_stats": {
    "queries_last_hour": 42,
    "avg_top_score": 0.82,
    "avg_chunks_after_threshold": 4.1,
    "queries_with_no_results": 3,
    "parameter_usage": {
      "top_k": {
        "min": 3,
        "max": 10,
        "median": 5
      },
      "similarity_threshold": {
        "min": 0.65,
        "max": 0.85,
        "median": 0.70
      }
    }
  }
}
```

---

## 6. Error Handling Patterns

### Decision: Layered Error Handling with User-Friendly Messages

**Recommended Pattern:**

```python
class RAGException(Exception):
    """Base exception for RAG operations"""
    def __init__(self, message: str, user_message: str, details: dict = None):
        self.message = message  # Technical details for logs
        self.user_message = user_message  # User-facing message
        self.details = details or {}
        super().__init__(message)

class DatabaseConnectionError(RAGException):
    """Vector database connection failed"""
    pass

class EmbeddingModelMismatchError(RAGException):
    """Embedding model incompatible with database"""
    pass

class LLMGenerationError(RAGException):
    """LLM generation failed"""
    pass

class ContextWindowExceededError(RAGException):
    """Context exceeds model's token limit"""
    pass


@app.exception_handler(RAGException)
async def rag_exception_handler(request: Request, exc: RAGException):
    """
    Convert RAG exceptions to user-friendly HTTP responses
    """
    logger.error(
        "RAG operation failed",
        error=exc.message,
        details=exc.details,
        endpoint=request.url.path
    )

    # Map to HTTP status codes
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


# Error handling in RAG pipeline
async def query_with_error_handling(query: str) -> RAGResponse:
    """Execute query with comprehensive error handling"""

    try:
        # 1. Validate query
        if not query or len(query.strip()) < 3:
            raise RAGException(
                message="Query too short",
                user_message="Please provide a query with at least 3 characters.",
                details={"query_length": len(query)}
            )

        # 2. Retrieve chunks
        try:
            chunks = await vector_store.similarity_search(query)
        except psycopg.OperationalError as e:
            raise DatabaseConnectionError(
                message=f"Database connection failed: {e}",
                user_message="Unable to connect to the document database. Please try again in a moment.",
                details={"db_error": str(e)}
            )

        # 3. Validate embedding compatibility
        if not validate_embedding_model(chunks):
            raise EmbeddingModelMismatchError(
                message="Embedding model mismatch detected",
                user_message=(
                    "The query embedding model doesn't match the document database. "
                    "Please check configuration or re-process documents."
                ),
                details={
                    "query_model": config.embedding_model,
                    "db_model": chunks[0].metadata.get("embedding_model")
                }
            )

        # 4. Assemble context
        try:
            context, selected_chunks = context_manager.assemble_context(
                chunks, query, system_prompt
            )
        except Exception as e:
            raise ContextWindowExceededError(
                message=f"Context assembly failed: {e}",
                user_message="The retrieved content is too long to process. Try a more specific query.",
                details={"chunks_attempted": len(chunks)}
            )

        # 5. Generate response
        try:
            response = await llm_client.generate(
                prompt=build_prompt(query, context),
                timeout=config.generation_timeout
            )
        except TimeoutError:
            raise LLMGenerationError(
                message="LLM generation timeout",
                user_message="Response generation took too long. The model may be overloaded.",
                details={"timeout": config.generation_timeout}
            )
        except Exception as e:
            raise LLMGenerationError(
                message=f"LLM generation failed: {e}",
                user_message="Unable to generate a response. Please try again.",
                details={"llm_error": str(e)}
            )

        return RAGResponse(
            answer=response.text,
            sources=selected_chunks,
            confidence="high"
        )

    except RAGException:
        raise  # Re-raise RAG exceptions (handled by exception handler)
    except Exception as e:
        # Catch-all for unexpected errors
        logger.exception("Unexpected error in RAG pipeline")
        raise RAGException(
            message=f"Unexpected error: {e}",
            user_message="An unexpected error occurred. Please try again or contact support.",
            details={"error_type": type(e).__name__}
        )


def get_error_suggestion(error_type: type) -> str:
    """Provide actionable suggestions for each error type"""
    suggestions = {
        DatabaseConnectionError: "Check that the PostgreSQL database is running and accessible.",
        EmbeddingModelMismatchError: "Verify that the embedding model in config.yaml matches the model used to process documents.",
        LLMGenerationError: "Check that the LLM server (RamaLama/Ollama) is running and the model is loaded.",
        ContextWindowExceededError: "Try a more specific query or reduce the top_k parameter.",
    }
    return suggestions.get(error_type, "Check system logs for more details.")
```

### Rationale

1. **User-Friendly Messages**: Separating technical details from user messages makes errors actionable for non-technical users while preserving debugging info.

2. **Structured Error Types**: Specific exception classes enable precise handling and status code mapping, making API behavior predictable.

3. **Actionable Suggestions**: Each error type includes specific remediation steps, reducing support burden and improving demo self-service UX.

4. **Logging Integration**: All errors logged with structured context enables post-mortem analysis and pattern detection.

5. **Progressive Detail**: Debug mode exposes technical details; production mode hides them. This makes demo suitable for both learning (debug on) and presentation (debug off) contexts.

### Configuration Parameters

```yaml
error_handling:
  debug_mode: true  # Include technical details in responses
  log_level: "INFO"
  include_suggestions: true

  timeouts:
    database_query: 5  # seconds
    llm_generation: 30  # seconds
    total_request: 35  # seconds

  retries:
    database_operations: 3
    llm_generation: 1  # Usually not worth retrying
    exponential_backoff: true
```

### Quality Considerations

**Health Check Integration:**
```python
@app.get("/health")
async def health_check():
    """Comprehensive health check for troubleshooting"""
    health = {
        "status": "healthy",
        "checks": {}
    }

    # Check database
    try:
        await vector_store.ping()
        health["checks"]["database"] = {"status": "ok"}
    except Exception as e:
        health["checks"]["database"] = {"status": "error", "message": str(e)}
        health["status"] = "unhealthy"

    # Check LLM
    try:
        await llm_client.ping()
        health["checks"]["llm"] = {"status": "ok"}
    except Exception as e:
        health["checks"]["llm"] = {"status": "error", "message": str(e)}
        health["status"] = "unhealthy"

    # Check embedding model
    try:
        validate_embedding_model_loaded()
        health["checks"]["embedding_model"] = {"status": "ok"}
    except Exception as e:
        health["checks"]["embedding_model"] = {"status": "error", "message": str(e)}
        health["status"] = "unhealthy"

    status_code = 200 if health["status"] == "healthy" else 503
    return JSONResponse(status_code=status_code, content=health)
```

**Error Recovery Patterns:**
```python
# Retry with exponential backoff
async def retry_with_backoff(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return await func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt  # 1s, 2s, 4s
            logger.warning(f"Retry {attempt+1}/{max_retries} after {wait_time}s", error=str(e))
            await asyncio.sleep(wait_time)
```

---

## Summary & Recommendations

### Recommended Pipeline Configuration

**Core Architecture:**
```yaml
pipeline:
  # Prompt Configuration
  prompts:
    system_prompt: "You are a helpful assistant..." (structured citation template)
    citation_format: "[Source: {source_file}, chunk {chunk_id}]"

  # Context Management
  context:
    max_tokens: 3000  # Adjust per model
    response_reserve: 500
    truncation_strategy: "priority_based"

  # Retrieval Parameters
  retrieval:
    top_k: 5
    similarity_threshold: 0.7
    runtime_override: true  # Allow API-level tuning

  # Response Mode
  response:
    streaming: true  # Primary mode
    batch: true  # Also provide for comparison
    include_citations_in_stream: true

  # Error Handling
  errors:
    debug_mode: true  # For demo
    user_friendly_messages: true
    include_suggestions: true
```

**Technology Stack:**
- **Orchestration**: Llamaindex (latest stable)
- **LLM Serving**: RamaLama/Ollama with Phi-4-mini or Qwen2.5:7b
- **Vector Store**: PostgreSQL + pgvector (existing docs2db schema)
- **API Framework**: FastAPI with async/await
- **Embedding**: Reuse model from docs2db pipeline (consistency critical)

### Implementation Priority

**Phase 1: Core Functionality**
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
- Citation accuracy: 100% (every statement must be cited)
- Relevance: Avg similarity score > 0.75 for included chunks
- Out-of-scope handling: 0% hallucination rate (return "not found" instead)

**Performance:**
- Query latency: <5s end-to-end (retrieve + generate)
- Context assembly: <500ms
- Streaming first-token: <1s

**Reliability:**
- Startup success rate: >95%
- Query success rate: >99% (excluding user errors)
- Graceful degradation: All error paths tested

### Lessons for Learners

This demo architecture teaches:
1. **Citation discipline**: RAG systems must attribute every claim
2. **Token budgeting**: Context windows are finite, require management
3. **Threshold tuning**: Similarity scores determine quality-relevance tradeoff
4. **Graceful failure**: Systems must explicitly handle out-of-scope queries
5. **Streaming UX**: Modern LLM apps stream for perceived performance
6. **Observable behavior**: Instrumentation enables understanding and tuning

### Alternative Approaches Documented

Include in demo documentation:
- **Batch vs. Streaming**: Show both, explain tradeoffs
- **Fixed vs. Runtime Parameters**: Demonstrate experimentation value
- **Strict vs. Permissive Thresholds**: Let users experience quality impact
- **Simple vs. Layered Errors**: Show evolution from minimal to production-quality

---

## References & Further Reading

**Llamaindex Documentation:**
- [Query Pipeline Architecture](https://docs.llamaindex.ai/en/stable/module_guides/querying/)
- [Citation Tracking](https://docs.llamaindex.ai/en/stable/examples/response_synthesis/citation.html)
- [Streaming Responses](https://docs.llamaindex.ai/en/stable/examples/response_synthesis/streaming_response.html)

**RAG Best Practices:**
- "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (Lewis et al., 2020)
- "Precise Zero-Shot Dense Retrieval without Relevance Labels" (Gao et al., 2023)
- [LangChain RAG Guide](https://python.langchain.com/docs/use_cases/question_answering/)

**Production Patterns:**
- [OpenAI RAG Best Practices](https://platform.openai.com/docs/guides/embeddings)
- [Anthropic Prompt Engineering](https://docs.anthropic.com/claude/docs/prompt-engineering)

**Local LLM Serving:**
- [Ollama Documentation](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [RamaLama Project](https://github.com/containers/ramalama)

---

**Document Status**: Complete
**Next Steps**: Use this research to design Phase 1 contracts and implementation plan
**Questions for Review**:
- Are default thresholds (0.7, top_k=5) appropriate for target corpus?
- Should demo include both streaming and batch endpoints, or streaming only?
- Is debug_mode=true appropriate for demo default, or should be toggled?
