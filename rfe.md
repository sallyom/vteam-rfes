# Llamaindex + RamaLama RAG Demo for docs2db-api

**Feature Overview:**

A containerized demonstration showcasing RAG (Retrieval-Augmented Generation) capabilities using Llamaindex orchestration, RamaLama model serving, and PGVectorDB created from user documents via the docs2db pipeline. This demo is designed for local development and testing using **Podman pods** (deployed via `podman kube play`), with a clear migration path to OpenShift for production deployments. The demo enables ML engineers, data scientists, and customers to experience production-like RAG workflows by converting their own document collections into an intelligent, queryable knowledge base. The feature provides a reference architecture for deploying Llamaindex-based RAG systems in containerized environments, following proven patterns from the existing llama-stack demo.

**Goals:**

* **Enable rapid RAG prototyping**: Provide ML engineers and data scientists with a working Llamaindex + RamaLama demo that can be deployed in under 30 minutes with their own documents using Podman on a local workstation
* **Demonstrate docs2db pipeline value**: Showcase the complete journey from raw documents (PDFs, HTML, Markdown) to a production-ready vector database suitable for RAG applications
* **Establish reference architecture**: Create a reusable pattern for deploying Llamaindex-based RAG systems using Podman pods locally, with straightforward migration to OpenShift for production
* **Reduce integration complexity**: Provide pre-configured containers, Kubernetes-compatible pod definitions, and configuration templates that handle the complexity of integrating Llamaindex with PGVector and RamaLama
* **Support customer POCs**: Give pre-sales and solution architects a demo they can run on their laptops with customer documents for proof-of-concept scenarios, then deploy to OpenShift when ready
* **Podman-first development**: Leverage `podman kube play` to use Kubernetes YAML for local development, ensuring seamless transition to OpenShift without rewriting deployment manifests

The difference between today's state and a world with this feature: Currently, users must manually integrate Llamaindex with vector databases, configure model serving, and handle embedding compatibility—requiring days of setup. With this demo, users get a working RAG system in minutes, learning best practices through a proven reference implementation.

**Out of Scope:**

* **Document ingestion within the demo**: The demo assumes PGVectorDB is pre-populated using the docs2db pipeline (ingest → chunk → embed → load); real-time document upload and processing is not included
* **Multi-tenancy support**: Single database instance, single embedding model per demo deployment
* **Authentication and authorization**: No security layer on API endpoints; suitable for demo/development environments only
* **Production hardening**: No rate limiting, advanced connection pooling, failover/HA configuration, or production monitoring
* **Advanced Llamaindex features**: No query transformations (HyDE, multi-step reasoning), fusion retrieval, or reranking; basic RAG pattern only (retrieve + generate)
* **Dynamic model switching**: Demo uses fixed embedding model and LLM; changing models requires pipeline re-run
* **Schema migration utilities**: No support for adapting non-docs2db vector databases to work with this demo
* **Full-featured UI**: If UI is included, it will be a basic query interface only (no document management, admin panel, or advanced analytics)
* **Existing database integration**: Demo does not support connecting to pre-existing PGVector databases not created by docs2db pipeline

**Requirements:**

**MVP Requirements** *(blocking for feature release)*:

1. **Podman Pod Architecture**: 4-container pod definition (pgvector, llamaindex-server, ramalama-model, optional UI) deployed using `podman kube play pod.yaml` with shared networking on localhost
2. **Pod Definition (YAML)**: Kubernetes-compatible `pod.yaml` that works with both `podman kube play` (local) and OpenShift (production), including:
   - Resource limits and requests
   - Liveness and readiness probes
   - Volume mounts for model storage (Podman volume / PVC)
   - Shared memory configuration for PostgreSQL
   - ConfigMap integration
3. **Container Builds**: Containerfiles for:
   - **pgvector**: PostgreSQL with pgvector extension, ragdb_dump.sql loading via entrypoint script
   - **llamaindex-server**: Python app with Llamaindex, custom vector store, psycopg2 dependency management (learned from llama-stack pattern)
   - Makefile targets for building images locally with Podman
4. **docs2db Integration**: PGVectorDB must be created using docs2db pipeline; demo includes clear instructions for running `docs2db ingest`, `chunk`, `embed`, and `load` commands to generate ragdb_dump.sql
5. **Custom Vector Store Adapter**: Llamaindex vector store implementation that reads docs2db database schema (documents, chunks, embeddings, models tables) without requiring schema migration
6. **Embedding Model Validation**: Startup validation that Llamaindex embedding model matches the model used in docs2db pipeline (reads from `models` table); fails-fast with clear error if mismatch detected
7. **Query API Endpoint**: RESTful API exposing `/query` endpoint for RAG queries, `/health` for readiness checks, and `/stats` for database statistics
8. **ConfigMap**: Llamaindex configuration YAML including database connection, embedding model, RamaLama endpoint, and similarity thresholds
9. **Deployment Scripts**: Convenience scripts or Makefile targets for:
   - Creating Podman volumes for model storage
   - Starting pod with `podman kube play`
   - Monitoring pod startup and health checks
   - Stopping and cleaning up pod
10. **README Documentation**: Quick start guide with focus on Podman workflow, prerequisites, service endpoints, testing instructions, troubleshooting section, and OpenShift migration guidance
11. **Example Queries**: At least 3 working example queries demonstrating successful document retrieval and answer generation

**Non-MVP Requirements** *(nice-to-have, can be deferred)*:

12. **Web UI**: Streamlit-based interface for interactive querying (similar to llama-stack demo UI)
13. **Multiple Model Support**: Configuration options for different embedding models (granite, all-MiniLM) and LLMs beyond Phi-4-mini
14. **Vector Index Optimization**: Scripts for creating HNSW or IVFFlat indexes on embedding vectors for improved query performance
15. **Performance Benchmarking**: Documentation of query latency and throughput for various dataset sizes (100 docs, 1k docs, 10k docs)
16. **OpenShift Deployment Guide**: Detailed documentation for migrating from Podman to OpenShift, including PVC configuration, ImageStreams, Routes, and resource quotas
17. **OpenShift-specific Manifests**: Optional Kubernetes Deployment/Service/Route resources for direct OpenShift deployment (alternative to pod.yaml)
18. **Monitoring Integration**: Prometheus metrics for query count, latency, and vector database size
19. **Advanced Configuration Examples**: Multiple ConfigMap templates for different use cases (high-precision vs. high-recall, different chunk sizes)
20. **Podman Compose Alternative**: Optional `podman-compose.yml` for users who prefer Docker Compose syntax over Kubernetes YAML

**Done - Acceptance Criteria:**

The feature is considered complete when:

* **Functional Completeness**:
  - [ ] Pod starts successfully with `podman kube play pod.yaml` command
  - [ ] All containers reach ready state within 2 minutes (readiness probes pass)
  - [ ] PGVector database loads from `ragdb_dump.sql` dump successfully on first startup
  - [ ] RamaLama model server serves the configured model (Phi-4-mini-instruct by default)
  - [ ] Llamaindex-server successfully connects to PGVector and validates embedding model compatibility
  - [ ] Query endpoint returns relevant results: Given a query about content in the indexed documents, the API returns responses with >0.7 similarity scores from correct source documents
  - [ ] Health check endpoint (`/health`) reports "healthy" status for all components (vector_db: connected, model: loaded)

* **Code Quality**:
  - [ ] All YAML, Containerfiles, and configuration files follow project conventions (consistent with llama-stack demo structure)
  - [ ] Resource limits defined and tested (minimum: 4GB RAM for pod, 20GB PVC for models)
  - [ ] No hardcoded secrets in code (database credentials in ConfigMap acceptable for demo purposes)
  - [ ] Custom vector store adapter includes comprehensive error handling and logging
  - [ ] Code passes peer review with at least one senior engineer approval

* **Documentation**:
  - [ ] README.md complete with: Overview, Prerequisites, Quick Start, Service Endpoints, Testing, Configuration, Troubleshooting, Cleanup
  - [ ] Architecture diagram showing 4-container pod structure and data flow
  - [ ] Quick Start verified by team member not involved in implementation (< 30 minute setup time)
  - [ ] Troubleshooting guide includes at least 5 common issues with resolutions
  - [ ] Example queries documented with expected outputs

* **Testing**:
  - [ ] Manual smoke test passes with documented test script
  - [ ] Integration test successful with sample documents from 3 different formats (PDF, HTML, Markdown)
  - [ ] Resource usage documented: CPU and memory utilization under typical query load
  - [ ] Embedding model mismatch scenario tested: System fails gracefully with clear error message
  - [ ] Cold start test: Pod restart and database reload completes successfully

* **Deployment**:
  - [ ] Demo structure matches existing pattern: `demos/llamaindex/` directory parallel to `demos/llama-stack/`
  - [ ] All build artifacts checked into repository: Containerfiles, pod.yaml, configmap.yaml, entrypoint scripts
  - [ ] No external dependencies beyond those already in docs2db-api project
  - [ ] Sample document set provided for testing (optional but recommended)

**Use Cases - i.e. User Experience & Workflow:**

**Primary Use Case: ML Engineer Evaluating RAG on OpenShift AI**

*Main Success Scenario:*

1. **Document Preparation**:
   - User has a collection of technical documents (product manuals, API docs, research papers) in a folder
   - User navigates to the folder: `/home/user/my-documents/`
   - Folder contains mixed formats: 15 PDFs, 10 HTML files, 5 Markdown files

2. **Run docs2db Pipeline**:
   - User executes: `make docs2db SOURCE=/home/user/my-documents/`
   - Pipeline runs automatically: ingest → chunk → embed → load
   - Output: `ragdb_dump.sql` file created (contains populated PGVectorDB)
   - Duration: ~5 minutes for 30 documents

3. **Build Demo Containers**:
   - User navigates to: `cd demos/llamaindex/`
   - User builds containers: `make build-containers`
   - Podman builds pgvector image (with ragdb_dump.sql embedded) and llamaindex-server image locally
   - Images tagged as `localhost/llamaindex-pgvector:latest` and `localhost/llamaindex-server:latest`
   - Duration: ~3 minutes

4. **Deploy Demo Pod**:
   - User creates Podman volume for models: `podman volume create ramalama-models`
   - User starts pod: `podman kube play pod.yaml` (or `make start-pod` for convenience)
   - Podman creates pod and starts 4 containers: pgvector, llamaindex-server, ramalama-model, ui
   - All containers share localhost networking (can communicate via 127.0.0.1)
   - User monitors startup: `podman pod logs -f llamaindex-pod`
   - Health checks confirm all services reach ready state
   - Duration: ~5 minutes (first run downloads Phi-4-mini model from HuggingFace)

5. **Query Knowledge Base**:
   - User accesses UI: `http://localhost:8501`
   - User enters query: "What are the system requirements for installation?"
   - System retrieves relevant chunks from PGVector, generates response using RamaLama
   - User sees: Generated answer + source citations with similarity scores
   - Response time: ~2-3 seconds

6. **Iterate and Refine**:
   - User tests additional queries from example list
   - User experiments with different similarity thresholds via API
   - User reviews architecture to understand integration patterns
   - User decides to adopt pattern for production use case

*Alternative Flow 1: API-only Usage (No UI)*

- After step 4, user directly queries API endpoint:
  ```bash
  curl -X POST http://localhost:8000/query \
    -H "Content-Type: application/json" \
    -d '{"query": "system requirements", "top_k": 5}'
  ```
- User integrates API into existing application/workflow

*Alternative Flow 2: Configuration Customization*

- Before step 4, user edits `configmap.yaml` to:
  - Change embedding model from all-MiniLM to granite-30m-english
  - Adjust similarity threshold from 0.7 to 0.8
  - Configure different LLM model
- User must re-run docs2db pipeline if embedding model changes
- User redeploys pod with updated ConfigMap

*Alternative Flow 3: Troubleshooting*

- Step 5 fails: Query returns empty results
- User checks embedding model match: `curl http://localhost:8000/stats`
- System reports: Mismatch detected (database has granite, Llamaindex using all-MiniLM)
- User corrects ConfigMap, restarts pod
- Query succeeds after fix

**Secondary Use Case: Solution Architect Preparing Customer Demo**

1. Solution architect receives customer documents (30 PDF contracts/policies)
2. Runs docs2db pipeline on customer documents
3. Deploys llamaindex demo with customer-specific database
4. Prepares 10 example queries relevant to customer use case
5. Demonstrates live during customer meeting
6. Customer sees their own content being queried intelligently
7. Leads to POC approval

**Use Case Diagram:**

```
┌─────────────────┐
│   ML Engineer   │
│  Data Scientist │
│ Solution Arch.  │
└────────┬────────┘
         │
         ├─────────────> [Prepare Documents]
         │                      │
         │                      v
         ├─────────────> [Run docs2db Pipeline]
         │                      │ (ingest/chunk/embed/load)
         │                      v
         │               [ragdb_dump.sql]
         │                      │
         ├─────────────> [Build Containers]
         │                      │
         │                      v
         ├─────────────> [Deploy Pod]
         │                      │
         │                      v
         │               ┌──────────────┐
         ├─────────────> │   Query API  │
         │               │   or Web UI  │
         │               └──────────────┘
         │                      │
         └─────────────> [Get RAG Results]
                                │
                                v
                    [Integrate or Iterate]
```

**Documentation Considerations:**

* **Quick Start Prerequisites**: Document must clearly state system requirements:
  - Minimum 4GB RAM available for pod
  - 20GB disk space for model storage
  - Podman 4.0+ installed and configured
  - docs2db installed and working (for pipeline execution)
  - Operating system: Linux (native), macOS (Podman machine), Windows (WSL2 + Podman)

* **Podman Deployment Workflow**: Provide step-by-step guide for:
  - Building container images with Podman
  - Creating Podman volumes for persistent storage
  - Deploying with `podman kube play pod.yaml`
  - Monitoring pod status and logs
  - Stopping pod with `podman kube play --down pod.yaml`
  - Cleanup procedures

* **docs2db Pipeline Integration**: Provide step-by-step guide for running docs2db pipeline, including troubleshooting common pipeline errors (out of memory, unsupported file formats)

* **Embedding Model Compatibility**: Dedicate section to explaining embedding model selection and the critical requirement for consistency between docs2db pipeline and Llamaindex configuration

* **Architecture Documentation**: Include visual diagram showing:
  - 4-container pod structure with shared localhost networking
  - Data flow from documents → docs2db pipeline → ragdb_dump.sql → PGVector → Llamaindex → RamaLama → Response
  - Port mappings (5432 for PostgreSQL, 8080 for RamaLama, 8000 for Llamaindex API, 8501 for UI)

* **Troubleshooting Guide**: Document at least 5 common failure scenarios:
  1. Embedding model mismatch (includes detection method and resolution)
  2. Container startup failures (memory limits, readiness probe timeouts)
  3. Database connection issues (psycopg2 import errors, learned from llama-stack)
  4. Model download failures (network issues, insufficient Podman volume space)
  5. Query returning no results (similarity threshold too high, empty database)
  6. Podman-specific issues (SELinux denials, volume mount permissions, network conflicts)

* **API Reference**: Document all endpoints with request/response schemas, example curl commands, and error codes

* **Configuration Reference**: Explain all ConfigMap parameters, acceptable value ranges, and performance implications of different settings

* **OpenShift Migration Path**: Document how to migrate from Podman to OpenShift:
  - Push container images to OpenShift registry or external registry
  - Convert Podman volumes to PersistentVolumeClaims
  - Add Route resources for external access
  - Adjust security contexts for OpenShift restricted SCC
  - Update ConfigMap for OpenShift-specific settings

* **Extension Points**: Document how to customize: add new endpoints, change models, integrate different vector stores, add authentication layer

* **Performance Tuning**: Provide guidance on vector index creation (HNSW/IVFFlat), batch size tuning, and expected query latency by dataset size

* **Comparison with llama-stack Demo**: Create comparison table showing differences in orchestration layer (Llamaindex vs Llama Stack), use cases, and when to choose each approach

**Questions to answer:**

*Architectural/Design Questions:*

1. **Vector Store Adapter Strategy**: Should we implement a custom `Docs2DBVectorStore` class that directly reads docs2db schema, OR create a migration utility that transforms docs2db schema into native Llamaindex schema? *(Recommendation: Custom adapter maintains docs2db as source of truth)*

2. **Embedding Model Loading Location**: Where should Llamaindex load the embedding model?
   - Option A: In llamaindex-server container (simpler, increases memory to ~2GB)
   - Option B: Separate embedding-server container (cleaner separation, more complex orchestration)
   *(Recommendation: Option A for MVP simplicity)*

3. **Metadata Contract**: docs2db chunks have JSONB metadata field - what metadata should be preserved and exposed in Llamaindex Node objects for retrieval/filtering? (source_file, chunk_index, processing parameters, etc.)

4. **Model Mismatch Handling**: When embedding model mismatch is detected, should the system:
   - Fail-fast with error (safer, forces correct configuration)
   - Warn and continue (more permissive, may produce poor results)
   *(Recommendation: Fail-fast for demo to prevent confusion)*

5. **Vector Index Strategy**: Should we include vector index creation (HNSW/IVFFlat) in the initial database setup, or document as optional performance optimization? Trade-off: index creation adds 30-60 seconds to startup but improves query latency 10-100x for large datasets.

*Implementation Questions:*

6. **Container Base Image**: Which base image for llamaindex-server? `python:3.11-slim` (minimal), `python:3.11` (includes more tools), or custom base with pre-installed system dependencies?

7. **Resource Limits**: What should be the default resource requests/limits in pod.yaml?
   - Memory: 4GB minimum observed, should we set 6GB limit for headroom?
   - CPU: 2 cores sufficient for typical queries?
   - Podman volume size: 20GB adequate for Phi-4-mini and embedding model?

8. **UI Inclusion**: Should Web UI be part of MVP or deferred to Phase 2? (impacts container count, complexity, and testing scope)

9. **Database Dump Loading**: Should we use the same `entrypoint.sh` pattern from llama-stack demo (auto-detect text/binary dump format), or require specific format?

10. **Error Handling Granularity**: For the custom vector store adapter, what level of error detail should be exposed to users? Full stack traces vs. user-friendly messages?

11. **Deployment Helper Scripts**: Should we provide:
    - Option A: Makefile targets only (`make start-pod`, `make stop-pod`, `make logs`)
    - Option B: Shell scripts (`./start.sh`, `./stop.sh`, `./status.sh`)
    - Option C: Both Makefile and scripts for flexibility
    *(Recommendation: Option C - Makefile for consistency with llama-stack demo, scripts for users unfamiliar with Make)*

12. **Podman Networking Mode**: Should pod use:
    - Option A: Infra container with shared localhost (default pod behavior)
    - Option B: Explicit port mappings from host to pod
    *(Recommendation: Option A for simplicity, matches Kubernetes behavior)*

*Testing & Validation Questions:*

13. **Sample Document Set**: Should we include a canonical sample document set in the repository for testing, or only provide instructions for users to bring their own documents?

14. **Performance Benchmarks**: What dataset sizes should we test and document? (100 docs / 1K docs / 10K docs / 100K docs?)

15. **Cross-Platform Testing**: Which Podman platforms must be validated before release?
    - Linux with native Podman (primary)
    - macOS with Podman machine (secondary)
    - Windows with WSL2 + Podman (nice-to-have)
    - OpenShift 4.x migration path (documented but not fully tested in MVP?)

*Documentation Questions:*

16. **Comparison Positioning**: How should we position this demo relative to llama-stack demo in documentation? Complementary alternatives or one supersedes the other?

17. **Troubleshooting Depth**: Should troubleshooting guide include container-level debugging (`podman logs`, `podman exec`) or stay at user-action level?

18. **OpenShift Documentation Scope**: For MVP, should OpenShift migration be:
    - High-level guidance only (key differences, conceptual mapping)
    - Detailed step-by-step instructions with example manifests
    *(Recommendation: High-level guidance in MVP, detailed guide as follow-up)*

**Background & Strategic Fit:**

This feature aligns with the strategic goal of establishing docs2db-api as a comprehensive RAG platform by providing multiple reference implementations for different orchestration frameworks. The existing llama-stack demo has proven valuable for users adopting Meta's Llama Stack framework, but the broader ML community has significant investment in Llamaindex, which has become a de facto standard for RAG orchestration.

**Market Context**: Llamaindex has over 35K GitHub stars and is widely adopted in enterprise RAG applications. Many customers evaluating OpenShift AI for RAG use cases specifically request Llamaindex examples because their teams already have Llamaindex expertise. By providing a Llamaindex demo alongside the llama-stack demo, we address a broader market and reduce friction for Llamaindex-experienced teams.

**Technical Context**: The docs2db pipeline provides a robust, production-quality path from documents to vector database. However, the consumption side (querying and generation) can be implemented with different orchestration frameworks. This demo demonstrates that docs2db is orchestration-agnostic and can integrate cleanly with any framework that supports PGVector.

**Podman-First Development Strategy**: Using `podman kube play` with Kubernetes YAML provides several strategic advantages:
- **Developer Velocity**: Developers can test locally on laptops without requiring OpenShift access, reducing iteration time from hours to minutes
- **Infrastructure Compatibility**: The same pod.yaml works with both Podman (local) and OpenShift (production), eliminating "works on my machine" deployment issues
- **Learning Curve**: Teams familiar with Kubernetes find Podman deployment intuitive, lowering adoption barriers
- **Cost Efficiency**: Local development with Podman eliminates cloud compute costs for demo/testing scenarios
- **Air-Gapped Environments**: Many enterprise customers work in restricted networks; Podman enables fully offline development and testing

**Competitive Landscape**: Cloud providers (AWS, Azure, GCP) provide managed RAG services but lack open-source, self-hosted alternatives. This demo showcases OpenShift AI's capability to run modern RAG stacks entirely on-premises or in private cloud environments, differentiating from cloud-only solutions. The Podman-first approach further differentiates by enabling local development without cloud dependencies.

**Internal Context**: This work builds on lessons learned from the llama-stack demo deployment, particularly around container orchestration patterns (psycopg2 dependency management, model volume handling, multi-container coordination). Reusing these patterns accelerates development and improves reliability. The llama-stack demo already uses `podman kube play`, validating this approach for multi-container RAG deployments.

**Customer Considerations**

* **Document Sensitivity**: Many customers (financial services, healthcare, government) cannot send documents to external LLM APIs due to compliance requirements. This demo's fully self-contained, on-premises architecture (RamaLama local model serving) addresses this concern directly.

* **Existing Llamaindex Investment**: Customers with existing Llamaindex-based applications can more easily adopt OpenShift AI if we demonstrate compatible patterns. Migration path should be clear: "If you're using Llamaindex + OpenAI embeddings today, here's how to run on OpenShift AI with docs2db + RamaLama."

* **Local-to-Production Workflow**: The Podman-first approach mirrors real-world customer workflows:
  1. Solution architect tests on laptop with sample documents (Podman)
  2. Validates with customer-specific documents locally (Podman)
  3. Demonstrates to customer stakeholders (Podman or OpenShift dev cluster)
  4. Deploys to customer's production OpenShift environment (same YAML)

  This workflow reduces risk and accelerates customer adoption by allowing iterative validation before production commitment.

* **Resource Constraints**: Not all customers have GPU infrastructure for model serving. The demo uses CPU-friendly models (Phi-4-mini quantized) to show that RAG is feasible without GPUs, though we should document GPU acceleration options for customers who have them. Podman local testing helps customers validate resource requirements (RAM, CPU, storage) before provisioning production infrastructure.

* **Multi-Model Strategy**: Some customers use different embedding models for different document types (legal vs. technical vs. financial). While MVP uses a single model, documentation should acknowledge multi-model scenarios and provide extension guidance.

* **Integration Requirements**: Customers will want to integrate RAG into existing applications. The API-first design (RESTful endpoints) enables integration with enterprise systems (ServiceNow, Salesforce, custom portals). Document common integration patterns.

* **Data Residency**: For international customers, data residency requirements may mandate that vector databases stay in specific regions/clusters. The containerized architecture supports distributed deployment; document multi-region considerations.

* **Audit and Compliance**: Some industries require audit trails for AI-generated responses. While MVP doesn't include audit logging, document extension points for adding: query logging, response provenance tracking, similarity score recording.

* **Performance Expectations**: Customers accustomed to cloud RAG services (sub-second responses) should understand self-hosted performance characteristics. Provide realistic performance benchmarks based on hardware specs (CPU vs GPU, memory, storage type).

* **Maintenance and Updates**: Customers need to understand the maintenance model: docs2db pipeline version compatibility, Llamaindex version pinning, model updates. Provide guidance on testing updates before production deployment.

* **Training and Onboarding**: Customer teams may need training on: docs2db pipeline operations, Llamaindex configuration, troubleshooting pod issues. Consider creating supplementary training materials or workshops based on this demo.
