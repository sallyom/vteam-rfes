# RamaLama Containerized Deployment Research

**Research Date**: 2025-10-30
**Context**: RAG Demo with Llamaindex + RamaLama
**Requirements**: Local LLM serving, CPU-only, 4GB RAM, <2min startup

---

## Executive Summary

### Decision: **RamaLama with llama.cpp backend serving quantized small language models**

**Rationale**:
- RamaLama provides containerized, secure-by-default LLM serving with automatic hardware detection
- llama.cpp backend offers optimized CPU inference with mature GGUF quantization support
- Container isolation aligns with production deployment patterns while maintaining demo simplicity
- OpenAI-compatible API enables seamless Llamaindex integration
- Established patterns for resource-constrained environments (4GB RAM)

**Trade-offs**:
- Inference speed will be constrained on CPU-only systems (1-4 tokens/second expected)
- Model selection limited to 1-3B parameter range for 4GB RAM constraint
- Initial container image pull adds ~2-5 minutes to first-run setup (cached thereafter)

---

## 1. RamaLama Architecture Overview

### What is RamaLama?

RamaLama is an open-source developer tool from the Containers organization that simplifies local AI model serving through containerization. It abstracts hardware detection, dependency management, and runtime configuration into a familiar container workflow.

**Core Capabilities**:
- Automatic hardware detection (GPU/CPU) with fallback strategies
- Multi-registry model support (Ollama, Hugging Face, OCI registries)
- Switchable inference runtimes (llama.cpp, vLLM, MLX)
- Security-first defaults (rootless containers, read-only model mounts, network isolation)
- OpenAI-compatible REST API for application integration

**Container Architecture**:
```
┌─────────────────────────────────────────┐
│ Host System                             │
│  ┌───────────────────────────────────┐  │
│  │ RamaLama CLI                      │  │
│  │ - Hardware detection              │  │
│  │ - Image selection                 │  │
│  │ - Container orchestration         │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│  ┌───────────▼───────────────────────┐  │
│  │ Podman/Docker Container           │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │ Runtime (llama.cpp/vLLM)    │  │  │
│  │  │ - Model loading             │  │  │
│  │  │ - Inference engine          │  │  │
│  │  │ - HTTP server (port 8080)   │  │  │
│  │  └─────────────────────────────┘  │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │ Model Volume (read-only)    │  │  │
│  │  │ - GGUF model file           │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**Security Posture**:
- Runs models in rootless containers (non-root user isolation)
- Network disabled by default (`--network=none` for run command)
- Models mounted read-only (prevents tampering)
- Container auto-cleanup on termination
- Process isolation from host system

### Hardware Detection & Image Selection

**Detection Flow**:
1. RamaLama inspects system for GPU support (NVIDIA CUDA, AMD ROCm, Intel GPU)
2. Falls back to CPU-only if no compatible GPU detected
3. Selects pre-configured OCI image from quay.io/ramalama optimized for detected hardware
4. Pulls container image with all necessary libraries/drivers (first run only)

**Container Images**:
- Base images: `quay.io/ramalama/<hardware>:<version>`
- Example: `quay.io/ramalama/cpu:1.2` for CPU-only systems
- RAG-enhanced images available with `-rag` suffix (includes Docling, Qdrant)
- Override default image with `--image` option

---

## 2. Container Deployment Configuration

### Basic Deployment Commands

**Interactive Chat Mode** (for testing):
```bash
ramalama run ollama://tinyllama:1.1b
```

**Server Mode** (for API access):
```bash
ramalama serve --port 8080 \
                --name demo-llm \
                ollama://tinyllama:1.1b
```

**Detached Server Mode** (background service):
```bash
ramalama serve -d \
                --port 8080 \
                --name demo-llm \
                ollama://tinyllama:1.1b
```

### CPU-Optimized Configuration

For 4GB RAM, CPU-only systems, configure explicit CPU inference with thread control:

```bash
ramalama serve --ngl 0 \
                --threads 4 \
                --ctx-size 2048 \
                --port 8080 \
                --name rag-demo-llm \
                ollama://tinyllama:1.1b-chat-q4_k_m
```

**Parameter Breakdown**:
- `--ngl 0`: Disable GPU layer offloading (force CPU-only)
- `--threads 4`: Use 4 CPU threads (default is half available cores; tune based on system)
- `--ctx-size 2048`: Context window of 2048 tokens (reduces memory footprint vs 4096 default)
- `--port 8080`: Serve on port 8080 (default, can be customized)
- `--name rag-demo-llm`: Container name for management commands

### Resource Management (Podman/Docker)

**Memory Limits** (via Podman/Docker):
```bash
# Run with explicit memory constraint
podman run --memory=3g --memory-swap=4g <ramalama-container>

# Or via RamaLama container engine passthrough (if supported)
RAMALAMA_CONTAINER_ENGINE=podman ramalama serve \
    --memory=3g \
    ollama://tinyllama:1.1b
```

**CPU Limits** (via Podman/Docker):
```bash
# Limit to 4 CPU cores
podman run --cpus=4 <ramalama-container>
```

**Note**: RamaLama abstracts container management, so direct Podman/Docker flags may require wrapper scripts or compose files for production deployments.

### Container Lifecycle Management

**Check Running Containers**:
```bash
ramalama ps
```

**Stop Container**:
```bash
ramalama stop rag-demo-llm
```

**Remove Container**:
```bash
ramalama rm rag-demo-llm
```

**List Pulled Models**:
```bash
ramalama list
```

**Monitor Performance** (via Podman/Docker):
```bash
podman stats <container-name>
```

---

## 3. Model Selection for CPU + 4GB RAM

### Recommended Models

Given **4GB RAM** and **CPU-only inference**, viable models are constrained to **1-3B parameter range** with **4-bit quantization (Q4_K_M)**.

#### **Primary Recommendation: TinyLlama 1.1B (Q4_K_M)**

**Model**: `ollama://tinyllama:1.1b-chat-q4_k_m`

**Characteristics**:
- Parameters: 1.1 billion
- Quantization: Q4_K_M (4-bit, medium quality)
- Memory footprint: ~1.7GB for model + ~1-1.5GB for inference overhead = ~3GB total
- Context window: 2048 tokens (configurable via `--ctx-size`)
- Training: 1 trillion tokens (OpenOrca dataset)

**Performance Expectations**:
- Inference speed: 1-4 tokens/second on modern CPU (2.5GHz+)
- Startup time: 30-60 seconds for model loading (after image pull)
- Suitable for: Demonstrations, simple Q&A, instruction following

**Trade-offs**:
- Limited reasoning capability vs larger models
- May struggle with complex multi-hop reasoning in RAG scenarios
- Acceptable quality degradation vs larger models for demo purposes

**RamaLama Command**:
```bash
ramalama serve --ngl 0 \
                --threads 4 \
                --ctx-size 2048 \
                --port 8080 \
                ollama://tinyllama:1.1b-chat-q4_k_m
```

#### **Alternative: Orca-Mini 3B (Q4_0)**

**Model**: `ollama://orca-mini:3b-q4_0`

**Characteristics**:
- Parameters: 3 billion
- Quantization: Q4_0 (4-bit, basic quality)
- Memory footprint: ~2.5GB for model + ~1-1.5GB overhead = ~4GB total (at limit)
- Better reasoning than TinyLlama, but higher memory pressure

**Use Case**: If TinyLlama quality insufficient and system can reliably provide 4GB+ RAM headroom.

**Caution**: This is the upper limit for 4GB systems. Performance may degrade under memory pressure.

#### **Fallback: Gemma 2B (Q4_K_M)**

**Model**: `ollama://gemma:2b-q4_k_m`

**Characteristics**:
- Parameters: 2 billion
- Quantization: Q4_K_M
- Memory footprint: ~1.7GB model + overhead = ~3GB total
- Google's efficient architecture optimized for resource constraints

**Use Case**: Balance between TinyLlama and Orca-Mini if quality/memory trade-off needed.

### Models to Avoid (Insufficient Resources)

**Mistral 7B**: Requires 8GB+ RAM even with Q4 quantization
**Llama 3.1 8B**: Requires 8GB+ RAM
**Phi-2**: Better quality but 2.7B params → ~3.5GB memory footprint, tight fit

### Quantization Format Explanation

**GGUF** (GPT-Generated Unified Format):
- Standard format for llama.cpp models
- Successor to GGML format
- Supports various quantization levels

**Q4_K_M** (Recommended):
- 4-bit quantization with k-quants (improved quality)
- "M" = medium quality variant (balances size/quality)
- ~4-5x smaller than original FP32 model
- Minimal quality degradation for models <7B params

**Quantization Trade-offs**:
- Q8_0: Best quality, 2x smaller than FP32, needs more RAM
- Q5_K_M: High quality, good balance, slightly larger than Q4
- Q4_K_M: **Recommended for 4GB RAM** - good quality, small size
- Q4_0: Smaller than Q4_K_M, lower quality
- Q3_K_M: Very small, noticeable quality loss
- Q2_K: Smallest, significant quality degradation

---

## 4. Health Check Implementation

### OpenAI-Compatible API Endpoints

RamaLama serves models via llama.cpp's HTTP server (or vLLM/MLX if specified), exposing OpenAI-compatible endpoints:

**Base URL**: `http://localhost:8080` (or configured port)

**Available Endpoints**:

1. **Chat Completions** (primary for RAG):
   ```
   POST /v1/chat/completions
   ```
   - OpenAI API-compatible
   - Accepts messages array, temperature, max_tokens, etc.
   - Returns streaming or non-streaming responses

2. **Completions** (legacy):
   ```
   POST /v1/completions
   ```
   - Text completion endpoint
   - Less commonly used for modern applications

3. **Models List**:
   ```
   GET /v1/models
   ```
   - Returns loaded model information
   - Can be used as basic health check

### Health Check Strategy

**Recommended Approach**: Poll `/v1/models` endpoint to verify service readiness.

**Health Check Implementation**:

```bash
# Simple health check (Bash)
curl -f http://localhost:8080/v1/models && echo "Service Ready" || echo "Service Not Ready"
```

```python
# Python health check
import requests
import time

def wait_for_ramalama(base_url="http://localhost:8080", timeout=120):
    """Wait for RamaLama service to become ready."""
    start_time = time.time()
    while time.time() - start_time < timeout:
        try:
            response = requests.get(f"{base_url}/v1/models", timeout=5)
            if response.status_code == 200:
                print("RamaLama service is ready")
                return True
        except requests.exceptions.RequestException:
            pass
        time.sleep(2)
    raise TimeoutError("RamaLama service did not become ready in time")
```

**Multi-Component Health Check** (for RAG demo):
```python
def check_rag_system_health():
    """Check all RAG system components."""
    checks = {
        "database": check_postgres_connection(),
        "embeddings": check_embedding_dimensions(),
        "llm": check_ramalama_ready(),
    }
    all_healthy = all(checks.values())
    return {"healthy": all_healthy, "components": checks}
```

### Startup Sequence Monitoring

**Expected Startup Timeline** (first run):
1. Container image pull: 2-5 minutes (one-time)
2. Container initialization: 5-10 seconds
3. Model loading: 30-60 seconds (depends on model size and disk I/O)
4. HTTP server ready: 2-5 seconds
5. **Total first run**: 3-6 minutes (image pull dominates)

**Expected Startup Timeline** (subsequent runs):
1. Container initialization: 5-10 seconds
2. Model loading: 30-60 seconds
3. HTTP server ready: 2-5 seconds
4. **Total**: 40-80 seconds

**Optimization**: Pre-pull container images in setup phase to meet <2 minute startup requirement for demos.

---

## 5. Integration with Llamaindex

### OpenAI API Compatibility

Llamaindex natively supports OpenAI-compatible APIs via the `OpenAILike` class.

**Configuration**:

```python
from llama_index.llms.openai_like import OpenAILike

# Configure RamaLama as LLM provider
llm = OpenAILike(
    api_base="http://localhost:8080/v1",
    api_key="dummy-key",  # RamaLama doesn't require auth
    model="tinyllama",    # Model name (informational)
    is_chat_model=True,   # Required for chat completions endpoint
    timeout=30.0,
    max_retries=3,
)
```

**Critical Parameters**:
- `is_chat_model=True`: **Required** to use `/v1/chat/completions` endpoint (will get 404 errors if False)
- `is_function_calling_model=True`: Set if using function calling features (agent workflows)
- `api_key`: Can be any value (RamaLama ignores authentication)

### RAG Pipeline Configuration

**Basic RAG Query Engine**:

```python
from llama_index.core import VectorStoreIndex, Settings

# Set global LLM
Settings.llm = llm

# Load documents from pgvector
vector_store = PGVectorStore.from_params(...)
index = VectorStoreIndex.from_vector_store(vector_store)

# Create query engine
query_engine = index.as_query_engine(
    similarity_top_k=5,
    response_mode="compact",
)

# Query
response = query_engine.query("What are the system requirements?")
```

**Performance Considerations**:
- Use `response_mode="compact"` to reduce token count in prompt
- Limit `similarity_top_k` to 3-5 chunks (avoid context overflow with small models)
- Enable streaming for better UX (`streaming=True`)

### Embedding Compatibility

**Important**: RamaLama serves **generation models only** (not embeddings). For RAG, you need separate embedding model.

**Recommended Approach**:
1. Use sentence-transformers for embeddings (e.g., `all-MiniLM-L6-v2`)
2. Store embeddings in PostgreSQL + pgvector
3. Use RamaLama only for generation step

**Embedding Model Selection**:
- `sentence-transformers/all-MiniLM-L6-v2`: 384 dimensions, 80MB, fast CPU inference
- `sentence-transformers/all-mpnet-base-v2`: 768 dimensions, 420MB, better quality

**Llamaindex Embedding Configuration**:
```python
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

embed_model = HuggingFaceEmbedding(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    cache_folder="./models",
)

Settings.embed_model = embed_model
```

---

## 6. Startup Optimization Techniques

### Pre-Deployment Optimizations

**1. Pre-Pull Container Images**:
```bash
# Pull image before demo deployment
ramalama pull ollama://tinyllama:1.1b-chat-q4_k_m
```
- Eliminates 2-5 minute image download from startup time
- Critical for meeting <2 minute startup requirement

**2. Pre-Download Models**:
```bash
# Models cached in ~/.local/share/ramalama
ramalama pull <model-name>
```
- Separates network I/O from startup sequence
- Enables fully offline demos

**3. Warm Container Start**:
- Keep container running in background
- Restart is faster than cold start
- Trade-off: Persistent resource consumption

### Runtime Optimizations

**1. Reduce Context Window**:
```bash
--ctx-size 2048  # Instead of 4096 default
```
- Reduces memory allocation
- Faster KV cache operations
- Sufficient for RAG use cases (retrieved context usually <1500 tokens)

**2. Optimize Thread Count**:
```bash
--threads 4  # Match physical cores, not hyperthreads
```
- More threads ≠ faster inference on CPU
- Optimal: Physical core count or half of logical cores
- Over-threading causes context switching overhead

**3. Batch Size Tuning**:
```bash
--batch-size 512  # Default is 512
```
- Affects prompt processing speed
- Higher = faster prompt ingestion, more memory
- For RAG: 512 is reasonable balance

**4. Model Loading Optimization**:
- Use SSDs for model storage (mechanical HDDs add 20-40s to load time)
- Place models on local disk, not NFS/network mounts

### Container-Level Optimizations

**1. Use Podman Machine with Appropriate Resources** (macOS/Windows):
```bash
podman machine stop
podman machine set --cpus 4 --memory 4096
podman machine start
```

**2. Disable Unnecessary Features**:
```bash
# Disable network for run command (serve needs network)
--network=none
```

**3. Use Volume Mounts for Models** (avoid re-downloads):
```bash
-v ~/.local/share/ramalama:/models:ro
```

### Monitoring & Tuning

**1. Track Startup Metrics**:
```bash
time ramalama serve ...
```
- Measure end-to-end startup time
- Identify bottlenecks (image pull, model load, server init)

**2. Monitor Memory Usage**:
```bash
podman stats <container-name>
```
- Verify staying within 4GB limit
- Detect memory leaks during long-running demos

**3. Log Analysis**:
```bash
podman logs <container-name>
```
- Check for model loading errors
- Verify HTTP server started
- Identify performance warnings

---

## 7. Performance Expectations

### Inference Latency

**TinyLlama 1.1B (Q4_K_M) on CPU**:
- Prompt processing: 50-100 tokens/second (batched)
- Token generation: 1-4 tokens/second (sequential)
- Time to first token: 2-5 seconds (includes retrieval + prompt processing)
- Full response (100 tokens): 25-100 seconds

**Factors Affecting Speed**:
- CPU speed (2.5GHz+ recommended)
- Memory bandwidth (DDR4 faster than DDR3)
- Thread configuration
- Context length (longer = slower due to KV cache)

### Throughput

**Single User**:
- Sequential queries only (no batching in llama.cpp server by default)
- 1 query per 30-120 seconds (depending on response length)

**Demo Constraints**:
- Not suitable for concurrent users (sequential processing)
- Acceptable for single-user demonstrations
- Production deployment would require GPU or multi-instance scaling

### Resource Utilization

**Memory**:
- Model: ~1.7GB (TinyLlama Q4_K_M)
- Inference overhead: ~1-1.5GB (KV cache, activations)
- Container overhead: ~200MB
- **Total**: ~3-3.5GB (within 4GB limit with headroom)

**CPU**:
- 80-100% utilization during generation (expected)
- 10-30% during idle (HTTP server)

**Disk I/O**:
- Model loading: ~1.7GB read (one-time per startup)
- Negligible during inference

**Network**:
- API calls: <1KB request, <10KB response (typical)
- No external network required for inference

---

## 8. Alternatives Considered

### Alternative 1: Ollama

**Description**: Similar to RamaLama but without container-first approach.

**Pros**:
- Mature ecosystem with extensive model library
- Simpler installation (native binary)
- Better documented OpenAI compatibility

**Cons**:
- Runs natively on host (less isolation)
- No built-in security features like RamaLama
- Harder to integrate with containerized workflows
- Less suitable for production deployment patterns

**Decision**: RamaLama chosen for container-first architecture alignment with production patterns.

### Alternative 2: llama.cpp Server Directly

**Description**: Run llama.cpp's built-in HTTP server without RamaLama wrapper.

**Pros**:
- Direct control over all parameters
- Minimal abstraction overhead
- Well-documented server options

**Cons**:
- Manual dependency management (build llama.cpp, install libraries)
- No automatic hardware optimization
- More complex setup for end users
- No multi-registry model support

**Decision**: RamaLama chosen for simplified setup and better demo experience.

### Alternative 3: vLLM (via RamaLama `--runtime=vllm`)

**Description**: Use vLLM inference engine instead of llama.cpp.

**Pros**:
- Higher throughput with continuous batching
- Better GPU utilization (if available)
- Production-grade serving features

**Cons**:
- Higher memory overhead (~500MB more)
- Longer startup time (~2x)
- Less optimized for CPU-only inference
- Overkill for demo purposes

**Decision**: llama.cpp chosen for CPU optimization and faster startup.

### Alternative 4: LocalAI

**Description**: Multi-backend local inference server (llama.cpp, vLLM, etc.).

**Pros**:
- Single server for multiple model types
- OpenAI-compatible API
- More features (TTS, image generation, etc.)

**Cons**:
- More complex configuration
- Larger resource footprint
- Additional dependencies
- Heavier than needed for demo

**Decision**: RamaLama chosen for simplicity and focus on LLM inference.

### Alternative 5: Cloud-Based LLM (OpenAI, Azure OpenAI)

**Description**: Use cloud provider's LLM API instead of local serving.

**Pros**:
- No local resource constraints
- Better model quality (GPT-4, etc.)
- Lower latency per token
- No startup time

**Cons**:
- Requires internet connectivity (violates "no cloud dependencies" requirement)
- Cost per query
- Data privacy concerns (documents sent to external service)
- Not suitable for edge/airgapped deployments

**Decision**: Rejected due to requirement for local, cloud-independent deployment.

---

## 9. Container Configuration Summary

### Recommended Compose File (for docs2db-api integration)

```yaml
version: '3.8'

services:
  ramalama-llm:
    image: quay.io/ramalama/cpu:latest  # Auto-selected by RamaLama
    container_name: rag-demo-llm
    command: >
      ramalama serve
        --ngl 0
        --threads 4
        --ctx-size 2048
        --port 8080
        ollama://tinyllama:1.1b-chat-q4_k_m
    ports:
      - "8080:8080"
    volumes:
      - ramalama-models:/root/.local/share/ramalama:ro
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 3.5G
        reservations:
          cpus: '2'
          memory: 2G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/v1/models"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 90s
    restart: unless-stopped

volumes:
  ramalama-models:
    driver: local
```

**Note**: This is a conceptual compose file. RamaLama abstracts container management, so actual deployment may use RamaLama CLI with wrapper scripts rather than direct compose files.

### Key Docker/Podman Settings

**Memory Limits**:
- Limit: 3.5GB (leaves headroom below 4GB constraint)
- Reservation: 2GB (guaranteed minimum)

**CPU Limits**:
- Limit: 4 cores (prevent resource starvation of other services)
- Reservation: 2 cores (minimum for acceptable performance)

**Health Check**:
- Command: Poll `/v1/models` endpoint
- Interval: 10s (check every 10 seconds)
- Start period: 90s (allow 90s for initial model loading)
- Retries: 5 (fail after 5 consecutive failures)

**Restart Policy**:
- `unless-stopped`: Auto-restart on failure, but honor manual stops
- Suitable for demo reliability without aggressive restarts

---

## 10. Integration Checklist for RAG Demo

### Pre-Deployment

- [ ] Install Podman or Docker on target system
- [ ] Pre-pull RamaLama container image: `ramalama pull ollama://tinyllama:1.1b-chat-q4_k_m`
- [ ] Verify 4GB+ RAM available: `free -h`
- [ ] Verify 20GB+ disk space available: `df -h`
- [ ] Test RamaLama installation: `ramalama --version`

### Deployment

- [ ] Start RamaLama server with CPU-optimized config
- [ ] Verify HTTP server listening on port 8080
- [ ] Test `/v1/models` endpoint returns 200 OK
- [ ] Test `/v1/chat/completions` with simple query
- [ ] Measure startup time (should be <2 minutes after image pre-pull)

### Llamaindex Integration

- [ ] Configure `OpenAILike` with `is_chat_model=True`
- [ ] Set `api_base="http://localhost:8080/v1"`
- [ ] Test basic query through Llamaindex
- [ ] Verify embedding model separate from generation model
- [ ] Validate embedding dimensions match pgvector schema

### Performance Validation

- [ ] Measure time-to-first-token (should be 2-5 seconds)
- [ ] Measure tokens per second (expect 1-4 t/s)
- [ ] Verify memory usage stays below 3.5GB during inference
- [ ] Test query with 5 retrieved chunks (context limit test)
- [ ] Measure end-to-end query time (should be <5 seconds to first token)

### Error Handling

- [ ] Test behavior when RamaLama service is down (connection refused)
- [ ] Test behavior when model is still loading (503 or timeout)
- [ ] Test behavior with empty database (no retrieval results)
- [ ] Test behavior with context overflow (query too long)
- [ ] Verify error messages are actionable

### Documentation

- [ ] Document RamaLama installation steps
- [ ] Document model selection rationale
- [ ] Document expected performance characteristics
- [ ] Document troubleshooting steps for common failures
- [ ] Document scaling path to production (GPU, larger models)

---

## 11. Recommended Next Steps

### For Implementation Team

1. **Phase 0 Validation**:
   - Deploy RamaLama with TinyLlama on target hardware
   - Benchmark inference speed and startup time
   - Confirm memory footprint within constraints

2. **Llamaindex Integration Prototype**:
   - Implement `OpenAILike` configuration
   - Test basic RAG pipeline with sample documents
   - Validate response quality with small model

3. **Health Check Implementation**:
   - Implement `/v1/models` polling with timeout
   - Add multi-component health check (DB + LLM + embeddings)
   - Test health check during startup sequence

4. **Performance Tuning**:
   - Benchmark different `--threads` values
   - Test different `--ctx-size` values
   - Optimize similarity_top_k for quality/speed balance

5. **Documentation**:
   - Write quickstart guide with RamaLama setup
   - Document expected vs. actual performance
   - Create troubleshooting guide

### For Architecture Review

**Trade-offs to Validate**:
- Is 1-4 tokens/second acceptable for demo purposes?
- Should we support Orca-Mini 3B as alternative (tighter memory fit)?
- Do we need streaming responses for better UX?
- Should health check be exposed as dedicated endpoint?

**Scaling Considerations**:
- Document path to GPU acceleration (same architecture, different `--ngl` value)
- Document path to larger models (Mistral 7B with 8GB+ RAM)
- Document multi-instance deployment for concurrent users
- Consider vLLM runtime for production (continuous batching)

---

## 12. References & Further Reading

**RamaLama Documentation**:
- GitHub: https://github.com/containers/ramalama
- Official Docs: https://docs.ramalama.com/
- Red Hat Developer Articles: https://developers.redhat.com/articles/2024/11/22/how-ramalama-makes-working-ai-models-boring

**llama.cpp Documentation**:
- GitHub: https://github.com/ggml-org/llama.cpp
- Server API: https://github.com/ggml-org/llama.cpp/blob/master/examples/server/README.md

**Llamaindex Documentation**:
- OpenAI-Like Integration: https://docs.llamaindex.ai/en/stable/examples/llm/openai_like/
- RAG Tutorial: https://docs.llamaindex.ai/en/stable/getting_started/starter_example/

**Model Selection Resources**:
- TinyLlama: https://huggingface.co/TinyLlama/TinyLlama-1.1B-Chat-v1.0
- Orca-Mini: https://www.hardware-corner.net/llm-database/Orca-Mini/
- Quantization Guide: https://towardsdatascience.com/quantize-llama-models-with-ggml-and-llama-cpp-3612dfbcc172

**Performance Benchmarks**:
- llama.cpp CPU Benchmarks: https://github.com/ggml-org/llama.cpp/discussions/638
- Low-Spec LLM Guide: https://matasoft.hr/qtrendcontrol/index.php/un-perplexed-spready/un-perplexed-spready-various-articles/145-leveraging-ai-on-low-spec-computers

---

## Appendix A: Quick Reference Commands

```bash
# Installation (assuming Podman/Docker installed)
pip install ramalama

# Pre-pull model
ramalama pull ollama://tinyllama:1.1b-chat-q4_k_m

# Start server (CPU-optimized, 4GB RAM)
ramalama serve --ngl 0 --threads 4 --ctx-size 2048 --port 8080 \
                ollama://tinyllama:1.1b-chat-q4_k_m

# Start server in background
ramalama serve -d --ngl 0 --threads 4 --ctx-size 2048 --port 8080 \
                --name rag-demo-llm \
                ollama://tinyllama:1.1b-chat-q4_k_m

# Health check
curl http://localhost:8080/v1/models

# Test chat completion
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "tinyllama",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'

# Stop server
ramalama stop rag-demo-llm

# Remove container
ramalama rm rag-demo-llm

# List models
ramalama list

# Monitor performance
podman stats rag-demo-llm
```

---

## Appendix B: Troubleshooting Guide

**Issue**: Container fails to start (port already in use)
```bash
# Check port availability
lsof -i :8080
# Kill conflicting process or use different port
ramalama serve --port 8081 ...
```

**Issue**: Model loading exceeds 2 minutes
```bash
# Pre-pull model before demo
ramalama pull <model-name>
# Check disk I/O speed
hdparm -t /dev/sda  # Linux
# Use SSD for model storage
```

**Issue**: Memory exceeded during inference
```bash
# Reduce context size
--ctx-size 1024  # Reduce from 2048
# Switch to smaller model
ollama://tinyllama:1.1b-chat-q4_0  # Q4_0 instead of Q4_K_M
```

**Issue**: 404 error on /v1/chat/completions
```bash
# Verify Llamaindex configuration
llm = OpenAILike(..., is_chat_model=True)  # Must be True
# Check RamaLama logs
podman logs <container-name>
```

**Issue**: Inference too slow (>10 seconds per token)
```bash
# Verify not swapping
free -h  # Check swap usage
# Reduce threads (over-threading overhead)
--threads 2  # Try lower thread count
# Check CPU throttling
grep MHz /proc/cpuinfo
```

---

**END OF RESEARCH DOCUMENT**
