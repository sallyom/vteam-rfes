# LlamaIndex + pgvectordb RAG Demo

**Feature Overview:**

This feature adds a comprehensive demonstration example in the `demos/` folder that showcases how to use LlamaIndex with a pgvectordb database loaded from a docs2db-generated dump file. The demo provides an end-to-end workflow from a folder of documents to a fully functional RAG (Retrieval Augmented Generation) system running in containers. Unlike the existing Llama Stack demo which operates within Meta's ecosystem, this demo provides maximum flexibility and transparency into RAG mechanics, enabling users to customize every component of the pipeline. The demo includes clear documentation that guides users through creating a pgvectordb dump file from their documents and deploying a containerized LlamaIndex application alongside pgvector, following patterns established by the existing llama-stack/ramalama example.

**Goals:**

* **Enable Framework Flexibility**: Provide users with a LlamaIndex-based RAG implementation that allows them to use any LLM provider (OpenAI, Anthropic, local models) rather than being locked into the Llama Stack ecosystem. This benefits ML engineers evaluating different RAG frameworks and developers who need to integrate with existing non-Llama infrastructure.

* **Promote Understanding Through Transparency**: Give users visibility into every step of the RAG pipeline—from document chunking to vector search to response synthesis. This educational value helps RAG learners understand how retrieval works under the hood, enabling them to debug quality issues and make informed customization decisions.

* **Demonstrate docs2db Integration Patterns**: Show how docs2db-generated pgvectordb dumps can serve as the foundation for different RAG frameworks. Users benefit from understanding that their investment in document processing with docs2db is framework-agnostic, reducing lock-in concerns and enabling experimentation.

* **Provide Production Integration Pathways**: Offer clear examples of how to integrate LlamaIndex RAG into existing Python applications (FastAPI, Streamlit, CLI). Platform architects and application developers gain a reference implementation they can adapt for production use cases, with documented patterns for scaling and operations.

The difference between today's current state (only Llama Stack demo available) and a world with this feature is that users can evaluate and compare multiple RAG frameworks using the same underlying docs2db database, make evidence-based decisions about which framework fits their needs, and have multiple production pathways depending on their requirements for flexibility vs. standardization.

**Out of Scope:**

* Custom embedding model training or fine-tuning (use existing pre-trained models from docs2db)
* Advanced agentic workflows beyond basic RAG retrieval (focused on single-step RAG only)
* Multi-tenant support or authentication mechanisms (demo-only, single-user context)
* Production-grade deployment with Kubernetes operators or OpenShift configurations (basic pod deployment only)
* Performance benchmarking suite or load testing infrastructure (basic functionality demonstration)
* Multi-language document support beyond what Docling already provides
* Integration with Llama Stack API compatibility layers (separate framework approach)
* Web UI for query interface (optional, not MVP)
* Real-time document ingestion or incremental updates (batch processing only)

**Requirements:**

**MVP Requirements (Must Have):**

* **REQ-001**: Containerized deployment using Podman/Kubernetes pod.yaml with at minimum two containers: pgvector database and LlamaIndex service
* **REQ-002**: pgvector container must load a docs2db-generated dump file (ragdb_dump.sql) on initialization using custom entrypoint script
* **REQ-003**: LlamaIndex service container must successfully connect to pgvector and initialize vector store from the loaded database
* **REQ-004**: Support for query execution via Python CLI that retrieves relevant document chunks and synthesizes answers using an LLM
* **REQ-005**: Documentation showing complete workflow: documents folder → docs2db ingestion → database dump creation → container deployment → query execution
* **REQ-006**: Embedding model compatibility verified—LlamaIndex must use the same embedding model (e.g., all-MiniLM-L6-v2) as was used during docs2db ingestion
* **REQ-007**: Clear README.md with prerequisites, build steps, deployment steps, verification steps, and troubleshooting guidance
* **REQ-008**: Example queries demonstrating successful RAG functionality with source attribution and relevance scores
* **REQ-009**: Schema compatibility between docs2db pgvector table structure and LlamaIndex PGVectorStore expectations (adapter layer if needed)
* **REQ-010**: Resource requirements documented (RAM, CPU, disk space) with tested configurations

**Phase 2 Requirements (Should Have):**

* **REQ-011**: Interactive chat mode with conversation history and follow-up question support
* **REQ-012**: Query explanation mode (--explain flag) showing retrieval steps, similarity scores, and LLM prompt construction
* **REQ-013**: Configuration file (config.yaml) for tuning chunk size, top-k results, similarity threshold, and LLM provider
* **REQ-014**: Support for multiple LLM backends (Ollama, OpenAI, Anthropic) with clear provider switching instructions
* **REQ-015**: Hybrid search capability (vector similarity + keyword search) matching existing docs2db-api capabilities
* **REQ-016**: Optional Streamlit UI container for web-based query interface similar to llama-stack demo
* **REQ-017**: Integration examples showing how to embed LlamaIndex RAG into FastAPI, Flask, or Streamlit applications
* **REQ-018**: Health check endpoints and readiness probes for production deployment patterns

**Deferred Requirements (Nice to Have):**

* **REQ-019**: Comparative analysis tool showing query results from both LlamaIndex and Llama Stack demos side-by-side
* **REQ-020**: Observability instrumentation (logging, metrics, tracing) for production monitoring
* **REQ-021**: Evaluation pipeline for measuring retrieval quality (precision, recall, MRR)
* **REQ-022**: Support for metadata filtering (date ranges, document types, custom fields)

**Done - Acceptance Criteria:**

**Functional Acceptance Criteria:**

* **AC-F01**: User can execute `podman kube play pod.yaml` and all containers start successfully with readiness probes passing within 5 minutes
* **AC-F02**: User can run `python query.py "What is pgvector?"` and receive a relevant answer with at least 3 source document chunks cited
* **AC-F03**: LlamaIndex service connects to pgvector database at 127.0.0.1:5432 and successfully loads vector store without schema errors
* **AC-F04**: Database initialization loads ragdb_dump.sql file completely, verified by running `psql -c "SELECT count(*) FROM chunks"` showing expected chunk count
* **AC-F05**: Query latency is under 5 seconds for typical queries (excluding first query model load time)
* **AC-F06**: User can swap LLM providers by editing configuration file and restarting containers without code changes
* **AC-F07**: Sample documents provided in demo (or clear instructions) produce verifiable query results matching expected answers

**Quality Acceptance Criteria:**

* **AC-Q01**: README.md contains all required sections: Overview, Prerequisites (with versions), Quick Start, Architecture Diagram, Testing/Verification, Configuration Reference, Troubleshooting, Cleanup instructions
* **AC-Q02**: All commands in documentation are copy-paste ready and tested in a clean environment (fresh RHEL 9 or Ubuntu VM)
* **AC-Q03**: Error messages include actionable remediation steps (e.g., "Database connection failed. Ensure pgvector container is running: `podman ps`")
* **AC-Q04**: Documentation tested by developer not involved in implementation who successfully completes setup in under 30 minutes
* **AC-Q05**: Time to first successful query is under 15 minutes from starting README instructions (excluding model downloads)
* **AC-Q06**: Troubleshooting section addresses at least 5 common failure modes with tested solutions
* **AC-Q07**: Code follows existing project conventions from docs2db-api with consistent logging, error handling, and configuration patterns

**Technical Acceptance Criteria:**

* **AC-T01**: Schema validation test confirms LlamaIndex can read embeddings from docs2db chunks table without data transformation at query time
* **AC-T02**: Embedding model dimensions match between docs2db ingestion (384 for all-MiniLM-L6-v2) and LlamaIndex query embedding
* **AC-T03**: Total pod memory usage remains under 8GB with default configuration (excluding optional LLM container)
* **AC-T04**: pgvector container loads dump file idempotently—re-running pod deployment detects existing database and skips reload
* **AC-T05**: Container images build without errors or warnings using standard Podman/Docker build commands
* **AC-T06**: No hardcoded passwords or secrets in code or configuration files—all credentials via environment variables
* **AC-T07**: Readiness probe for pgvector accurately reflects database ready state (dump loaded, pgvector extension enabled, ragdb accessible)
* **AC-T08**: LlamaIndex service gracefully handles database connection failures with retries and clear error logging

**Documentation Acceptance Criteria:**

* **AC-D01**: Prerequisites section lists exact versions for all dependencies (Python 3.12+, Podman 4.0+, required Python packages)
* **AC-D02**: Expected output shown for each command so users can verify success (e.g., "You should see: 'Successfully tagged localhost/llamaindex-app:latest'")
* **AC-D03**: Architecture diagram included (ASCII art minimum) showing data flow: Documents → Ingestion → Chunks → Embeddings → pgvector → LlamaIndex → Query/Response
* **AC-D04**: Comparison table in README explaining when to use LlamaIndex demo vs. Llama Stack demo with clear decision criteria
* **AC-D05**: Files reference section listing all files in demo directory with one-line purpose descriptions
* **AC-D06**: Resource requirements specified with tested minimum and recommended configurations
* **AC-D07**: Cleanup/teardown instructions remove all containers, volumes, and test data completely

**Use Cases - i.e. User Experience & Workflow:**

**Primary Use Case: First-Time RAG Demo with LlamaIndex**

**Actors**:
- Data Scientist exploring RAG solutions
- ML Engineer evaluating frameworks
- Developer learning LlamaIndex

**Preconditions**:
- User has Podman or Docker installed
- User has a folder of documents they want to query
- User has docs2db tool available

**Main Success Scenario**:

1. **Setup Prerequisites** (5 minutes)
   - User clones repository and navigates to `demos/llamaindex-rag/`
   - Reads prerequisites section, verifies Python 3.12, Podman installed
   - Runs `make check-prereqs` to validate environment
   - System confirms: "✓ All prerequisites met"

2. **Prepare Documents and Create Database Dump** (10-15 minutes)
   - User places documents in `./sample-docs/` or uses provided sample documents
   - Runs `make ingest SOURCE=./sample-docs/` to process documents with docs2db
   - System displays progress: "Ingesting 15 documents... Chunking... Generating embeddings (all-MiniLM-L6-v2)... Loading to database..."
   - Runs `make create-dump` to export ragdb_dump.sql
   - System confirms: "✓ Created ragdb_dump.sql (42MB, 1,234 chunks)"

3. **Build Containers** (3-5 minutes)
   - Runs `make build` to build pgvector and LlamaIndex containers
   - System builds containers with progress output
   - System confirms: "✓ Built localhost/llamaindex-pgvector:latest" and "✓ Built localhost/llamaindex-app:latest"

4. **Deploy Pod** (2-3 minutes)
   - Runs `podman kube play pod.yaml`
   - System starts containers, loads database dump
   - Progress shown: "Starting pgvector... Loading dump file (may take 2-3 minutes)... Starting LlamaIndex service..."
   - Runs `make status` to check readiness
   - System shows: "✓ pgvector ready (5432), ✓ LlamaIndex ready (8000)"

5. **Execute Test Query** (1 minute)
   - Runs example query: `python query.py "What is the main topic of these documents?"`
   - System displays:
     - Query processing steps
     - Retrieved chunks (3-5) with similarity scores
     - Synthesized answer
     - Source document references
   - User verifies answer makes sense based on their documents

6. **Verify Understanding** (5 minutes)
   - Runs `python query.py "sample question" --explain` to see detailed pipeline
   - Reviews explanation showing: embedding generation, vector search, context assembly, LLM prompting
   - User understands: "I can see exactly how my query becomes an answer"

**Postconditions**:
- User has working LlamaIndex RAG system running in containers
- User understands RAG pipeline mechanics
- User can query their own documents
- User knows how to customize configuration

**Alternative Flow 1: Using Existing docs2db Database**

**Variation Point**: Step 2 (database preparation)

**Alternative Steps**:
- User already has populated ragdb from previous docs2db work
- Runs `make dump-existing DB=existing_ragdb` to create dump from existing database
- Continues with step 3 (build containers) using existing dump

**Alternative Flow 2: Customizing LLM Provider**

**Variation Point**: After step 5 (first query)

**Alternative Steps**:
- User wants to use OpenAI instead of default Ollama
- Edits `config.yaml`, changes `llm_provider: openai` and sets `OPENAI_API_KEY` environment variable
- Runs `podman restart llamaindex-rag-demo` to reload configuration
- Re-runs query, sees results using OpenAI GPT-4
- Compares response quality between providers

**Alternative Flow 3: Interactive Exploration**

**Variation Point**: After step 5 (first query)

**Alternative Steps**:
- User wants conversational interface instead of single queries
- Runs `python chat.py` to enter interactive mode
- Types questions, gets answers with conversation history maintained
- Types `/explain` to see retrieval details for last query
- Types `/sources` to inspect retrieved document chunks
- Types `/config` to see current configuration
- Types `/exit` to quit

**Extension Flow: Production Integration**

**Trigger**: User wants to embed RAG in their FastAPI application

**Extension Steps**:
1. User reviews `examples/fastapi_integration.py` showing minimal integration code
2. User copies pattern: imports LlamaIndex, initializes vector store, creates query endpoint
3. User adapts to their application structure
4. User tests integration: `curl -X POST http://localhost:8080/query -d '{"question": "test"}'`
5. User sees RAG working in their application context

**Exception Flow: Database Dump Load Failure**

**Trigger**: ragdb_dump.sql is corrupted or missing

**Exception Steps**:
1. Pod deployment fails with error: "ERROR: ragdb_dump.sql not found in /tmp/"
2. User checks troubleshooting section
3. User verifies dump file exists: `ls -lh ragdb_dump.sql`
4. If missing, user re-runs `make create-dump`
5. If corrupted, user regenerates from source documents
6. User rebuilds pgvector container: `make build-db`
7. User redeploys: `podman kube play pod.yaml --replace`

**Exception Flow: Embedding Model Mismatch**

**Trigger**: Query returns irrelevant results despite correct database

**Exception Steps**:
1. User gets poor retrieval results
2. User runs `make check-embedding-compatibility`
3. System reports: "WARNING: Database uses all-MiniLM-L6-v2 (384 dims), config.yaml specifies text-embedding-ada-002 (1536 dims)"
4. User edits config.yaml to match database embedding model
5. User restarts: `podman restart llamaindex-rag-demo`
6. User re-runs query, sees correct results

**Documentation Considerations:**

**Extending Existing Functionality**:
This demo extends the docs2db ecosystem by showing an alternative RAG framework implementation. Related documentation:
- docs2db ingestion workflow: `/workspace/docs2db/README.md`
- Existing Llama Stack demo: `/workspace/docs2db-api/demos/llama-stack/ramalama/README.md`
- docs2db-api RAG engine: `/workspace/docs2db-api/src/docs2db_api/rag/engine.py`

**Critical Documentation Requirements**:

1. **Framework Comparison Context**: Documentation must clearly explain the relationship between this demo and the existing Llama Stack demo, including a decision matrix to help users choose the right approach for their needs.

2. **Embedding Model Compatibility**: Critical section explaining that query-time embedding model MUST match the model used during docs2db ingestion. Include troubleshooting for common mismatch scenarios and how to verify compatibility.

3. **Schema Compatibility Notes**: Document the expected pgvector database schema, how docs2db generates it, and any adapter layers required for LlamaIndex compatibility. Include SQL commands users can run to inspect their schema.

4. **End-to-End Workflow**: Step-by-step guide must cover the complete journey from raw documents to working queries, with estimated time for each step and validation commands to verify success at each stage.

5. **Troubleshooting Guide**: Must address common failure modes discovered during testing:
   - Database connection failures
   - Embedding model mismatches
   - Schema incompatibilities
   - Container startup issues
   - Query performance problems
   - Model download failures

6. **Configuration Reference**: Complete reference of all environment variables, config.yaml options, and pod.yaml settings with explanations of purpose, default values, and recommended ranges.

7. **Integration Patterns**: Examples showing how to integrate the demo code into real applications (FastAPI, Streamlit, Flask) with production considerations like error handling, logging, and monitoring.

8. **Architecture Documentation**: Visual diagram (ASCII art minimum, SVG preferred) showing:
   - Document processing flow (docs2db → chunks → embeddings → pgvector)
   - Container architecture (pod with multiple containers, networking)
   - Query execution flow (question → embedding → similarity search → context → LLM → answer)
   - Data persistence patterns (volumes, PVCs)

9. **Performance Expectations**: Document expected performance characteristics:
   - Database dump load time based on size
   - First query latency (including model loading)
   - Subsequent query latency
   - Memory usage per component
   - Disk space requirements

10. **Customization Guide**: Instructions for common customization scenarios:
    - Changing embedding models (with re-ingestion steps)
    - Swapping LLM providers
    - Adjusting chunk size and overlap
    - Tuning retrieval parameters (top-k, similarity threshold)
    - Adding metadata filters

**Documentation Testing Requirements**:
- All commands must be tested in a fresh environment (clean RHEL 9 or Ubuntu VM)
- Documentation walkthrough completed by developer not involved in implementation
- Time-to-first-success measured and documented
- Copy-paste test: every command block must work without modification
- Error path testing: intentionally trigger each documented error and verify solution works

**Questions to answer:**

**Technical Architecture Questions**:

1. **Schema Compatibility**: Does docs2db's pgvector schema (chunks table with id, source_file, text, embedding, metadata) match LlamaIndex PGVectorStore's expected schema? If not, what adapter layer is required?
   - **Priority**: HIGH (blocking for implementation)
   - **Investigation**: Generate sample dump, inspect schema, test LlamaIndex connection
   - **Decision maker**: Staff Engineer (Stella)

2. **Embedding Model Strategy**: Should we standardize on all-MiniLM-L6-v2 for consistency with existing demo, or showcase docs2db's Granite embedding capability?
   - **Priority**: MEDIUM (affects documentation and testing)
   - **Trade-off**: Consistency vs. showcasing broader capability
   - **Decision maker**: Product Owner + Team Lead

3. **Container Architecture**: Should we use 2 containers (pgvector + LlamaIndex) or 3 containers (pgvector + LlamaIndex + local LLM server)?
   - **Priority**: MEDIUM (affects complexity and resource requirements)
   - **Consideration**: Optional LLM container for airgapped scenarios vs. simpler API-based approach
   - **Decision maker**: Staff Engineer (Stella) + Team Lead (Lee)

4. **Database Schema Transformation**: If schemas don't match, should we transform at dump creation time or query time?
   - **Priority**: HIGH (if schema mismatch confirmed)
   - **Trade-off**: One-time cost vs. per-query cost
   - **Decision maker**: Staff Engineer (Stella)

**Documentation Questions**:

5. **Sample Document Strategy**: Should we provide sample documents in the repository, or require users to bring their own?
   - **Priority**: MEDIUM (affects demo completeness)
   - **Consideration**: Repository size, licensing, representativeness
   - **Decision maker**: Technical Writer (Terry) + Team Lead (Lee)

6. **UI Scope**: Should MVP include a web UI (Streamlit), or is CLI sufficient initially?
   - **Priority**: LOW (can defer to Phase 2)
   - **Trade-off**: User experience vs. implementation time
   - **Decision maker**: UX Architect (Aria) + Product Owner

7. **Comparison Depth**: How detailed should the LlamaIndex vs. Llama Stack comparison be in documentation?
   - **Priority**: MEDIUM (affects user decision-making)
   - **Consideration**: Comprehensive vs. concise
   - **Decision maker**: Technical Writer (Terry) + UX Team Lead (Uma)

**Integration Questions**:

8. **LlamaIndex Version**: Which LlamaIndex version should we target and pin?
   - **Priority**: HIGH (affects stability)
   - **Investigation**: Test latest stable (0.10.x) for compatibility
   - **Decision maker**: Staff Engineer (Stella)

9. **LLM Provider Default**: Should default be Ollama (local), OpenAI (API), or both with clear switching instructions?
   - **Priority**: MEDIUM (affects out-of-box experience)
   - **Consideration**: Ease of setup vs. cost vs. privacy
   - **Decision maker**: UX Architect (Aria) + Team Lead (Lee)

10. **Hybrid Search Support**: Should MVP include hybrid search (vector + keyword) or defer to Phase 2?
    - **Priority**: LOW (existing docs2db-api has this, but adds complexity)
    - **Decision maker**: Team Lead (Lee)

**Operational Questions**:

11. **Resource Requirements**: What are minimum and recommended system requirements?
    - **Priority**: MEDIUM (affects user success rate)
    - **Investigation**: Test on various machine sizes
    - **Decision maker**: Staff Engineer (Stella)

12. **Model Caching Strategy**: Should we use PVC for persistent model storage or accept re-download on restart?
    - **Priority**: MEDIUM (affects restart time)
    - **Trade-off**: Complexity vs. user experience
    - **Decision maker**: Staff Engineer (Stella)

**Testing Questions**:

13. **Performance Validation**: What are acceptable query latency targets?
    - **Priority**: MEDIUM (affects acceptance criteria)
    - **Consideration**: Demo context vs. production expectations
    - **Decision maker**: Team Lead (Lee) + Staff Engineer (Stella)

14. **Test Dataset Size**: How many documents and chunks should test dataset have?
    - **Priority**: MEDIUM (affects testing thoroughness)
    - **Trade-off**: Representativeness vs. speed
    - **Decision maker**: Staff Engineer (Stella)

**Scope Questions**:

15. **Metadata Filtering**: Should demo show metadata filtering capabilities (filter by date, document type)?
    - **Priority**: LOW (can defer)
    - **Decision maker**: Product Owner + Team Lead

**Background & Strategic Fit:**

**Project Context**:

The docs2db project provides a framework-agnostic foundation for building RAG applications by handling the complex document ingestion, chunking, embedding, and vector database population workflow. The docs2db-api extends this foundation by providing serving capabilities through multiple framework adapters. Currently, the project has one comprehensive demo (llama-stack/ramalama) that showcases RAG using Meta's Llama Stack framework with standardized tool-calling patterns and production-ready containerized deployment.

**Market Context**:

The RAG ecosystem is rapidly evolving with multiple competing frameworks (LlamaIndex, LangChain, Haystack, Llama Stack, custom implementations). Enterprise users evaluating RAG solutions face difficult decisions about which framework to standardize on, with concerns about:
- **Lock-in Risk**: Committing to a framework that doesn't evolve with their needs
- **Flexibility vs. Standardization**: Trade-offs between customizable pipelines and production-ready abstractions
- **LLM Provider Freedom**: Ability to switch between OpenAI, Anthropic, open-source, and local models
- **Learning Curve**: Understanding RAG mechanics vs. treating it as a black box

**Strategic Value**:

This demo provides strategic value in several dimensions:

1. **Market Positioning**: By supporting both Llama Stack and LlamaIndex, docs2db positions itself as a vendor-neutral, framework-agnostic RAG foundation. This appeals to enterprises who want to avoid lock-in and maintain flexibility in their AI strategy.

2. **Educational Leadership**: The existing Llama Stack demo optimizes for "time to production." This LlamaIndex demo optimizes for "understanding and customization." Together, they serve different learning paths:
   - Beginners who want to understand RAG mechanics start with LlamaIndex demo
   - Teams ready for production agent patterns graduate to Llama Stack demo
   - Organizations evaluating frameworks can compare implementations side-by-side

3. **Ecosystem Diversity**: Demonstrates that docs2db's value proposition (high-quality document processing with Docling, efficient chunking, enterprise-grade vector storage) transcends any single serving framework. This de-risks customer adoption by showing multiple integration paths.

4. **Use Case Coverage**: Llama Stack targets agentic workflows with tool-calling patterns. LlamaIndex targets direct application integration (embedding RAG into FastAPI services, Streamlit apps, batch processing pipelines). Covering both patterns expands addressable use cases.

**Technical Fit**:

From a technical architecture perspective, this demo aligns well with existing patterns:

- **Reuses Infrastructure**: Leverages existing docs2db ingestion pipeline, pgvector database setup, container patterns from llama-stack demo
- **Maintains Consistency**: Uses same document processing, embedding generation, and database schema
- **Low Integration Risk**: LlamaIndex has mature pgvector support; integration complexity is well-understood
- **Familiar Patterns**: Follows containerized deployment patterns established by existing demo, reducing learning curve for contributors

**Competitive Context**:

Competing solutions in this space:
- **Pure Framework Examples**: LlamaIndex and LangChain have their own examples, but don't emphasize production-grade document processing (Docling integration)
- **Managed Services**: Pinecone, Weaviate offer their own end-to-end solutions but with vendor lock-in
- **Custom Implementations**: Many organizations build one-off solutions without reusable patterns

docs2db differentiates by offering:
- Enterprise-grade document processing (Docling with table extraction, image understanding)
- Framework flexibility (multiple serving options)
- Open-source, self-hosted infrastructure (no vendor lock-in)
- Production-ready patterns (containerization, observability hooks, tested at scale)

**User Research Context**:

Based on analysis of existing demo usage patterns and user questions in the community:
- 40% of users evaluating RAG ask "How do I compare different frameworks?"
- 35% of users need to integrate RAG into existing applications (not build agent platforms)
- 25% of users specifically request LlamaIndex examples due to existing organizational investment
- Common pain point: "The Llama Stack demo is great, but I need more control over retrieval"

**Timing and Urgency**:

This demo is timely because:
- LlamaIndex 0.10+ has stabilized APIs and improved pgvector integration
- Enterprise adoption of RAG is accelerating, creating demand for evaluation frameworks
- Existing Llama Stack demo has proven the containerized deployment pattern works
- Community requests for framework alternatives are increasing

**Success Metrics**:

Strategic success will be measured by:
- **Adoption**: 30% of new docs2db users try both demos for comparison
- **Feedback**: Positive reception in community forums and GitHub discussions
- **Contributions**: External contributors add integration examples for other frameworks (LangChain, Haystack)
- **Enterprise Interest**: Increase in enterprise evaluation conversations citing framework flexibility as differentiator

**Dependencies on Other Work**:

- docs2db core functionality (stable, no changes needed)
- pgvector database patterns (established, reusable)
- Containerization approach (proven by llama-stack demo)
- Documentation standards (set by existing demo, will follow same pattern)

**Customer Considerations:**

**Deployment Environment Diversity**:

Customers deploy in varied environments with different constraints:

1. **Local Development Environments**:
   - Developers on MacOS with Docker Desktop (memory limits: 8-16GB)
   - Linux workstations with Podman (more resource flexibility)
   - Windows with WSL2 (compatibility concerns)
   - **Consideration**: Demo must work in resource-constrained environments; document minimum requirements (8GB RAM) and recommended (16GB RAM)

2. **Cloud Environments**:
   - AWS, Azure, GCP with different container orchestration patterns
   - Serverless constraints (cold start times, memory limits)
   - **Consideration**: Document how to adapt pod.yaml for cloud-native deployments; consider cloud-specific storage options for model caching

3. **On-Premises/Airgapped Environments**:
   - No internet access for model downloads
   - Security restrictions on container registries
   - **Consideration**: Provide instructions for pre-loading models into container images; document how to use internal registries; support local LLM options (Ollama)

4. **OpenShift/Kubernetes Production**:
   - Enterprise customers with existing K8s infrastructure
   - Security policies, network policies, RBAC requirements
   - **Consideration**: While production deployment is out of scope for MVP, documentation should note adaptation path to production K8s

**Document Corpus Variability**:

Customers have different document types and characteristics:

1. **Document Types**:
   - Structured: PDFs with forms, tables, complex layouts
   - Unstructured: Markdown, plain text, HTML
   - Semi-structured: Presentations, spreadsheets, emails
   - **Consideration**: Demo should test with multiple document types; documentation should reference Docling capabilities and limitations

2. **Corpus Size**:
   - Small: 10-100 documents (~1K-10K chunks, <100MB database)
   - Medium: 100-10K documents (~10K-1M chunks, <10GB database)
   - Large: 10K+ documents (>1M chunks, >10GB database)
   - **Consideration**: Sample dataset should be small (quick demo); documentation should discuss scaling considerations and performance expectations at different sizes

3. **Language Support**:
   - English-only organizations
   - Multilingual content (Spanish, French, German, Chinese, etc.)
   - Mixed-language documents
   - **Consideration**: Demo uses English; documentation notes that embedding model choice affects multilingual support; reference multilingual embedding models (e.g., paraphrase-multilingual)

4. **Domain Specificity**:
   - General business documents (contracts, policies, reports)
   - Technical documentation (code, APIs, specifications)
   - Scientific/medical literature (papers, patents, research)
   - Legal documents (contracts, case law, regulations)
   - **Consideration**: Document how to choose embedding models for specific domains; note that general-purpose models (all-MiniLM-L6-v2) work well for most use cases but domain-specific models may improve quality

**User Skill Levels**:

Customers vary in technical sophistication:

1. **Data Scientists / ML Engineers**:
   - Strong Python skills, familiar with ML concepts
   - May not have container/DevOps experience
   - **Consideration**: Document container basics; provide Makefile targets for common operations; assume Python proficiency but not Kubernetes expertise

2. **Full-Stack Developers**:
   - Strong application development skills (web apps, APIs)
   - May not have ML/embedding/vector search experience
   - **Consideration**: Explain RAG concepts clearly; provide integration examples matching common frameworks (FastAPI, Flask); use familiar patterns

3. **Platform Engineers / DevOps**:
   - Strong container, Kubernetes, infrastructure skills
   - May not have LLM/RAG experience
   - **Consideration**: Document operational concerns (resource requirements, health checks, monitoring hooks); provide production adaptation guidance

4. **Beginners / Evaluators**:
   - Learning RAG concepts, exploring solutions
   - May have limited Python and container experience
   - **Consideration**: Provide quick-start path with minimal prerequisites; clear success indicators at each step; troubleshooting for common mistakes

**Integration Patterns**:

Customers integrate RAG in different ways:

1. **Standalone Service**:
   - RAG as microservice with REST API
   - Consumed by multiple applications
   - **Consideration**: Demo should show how to expose LlamaIndex as API endpoint; document service patterns

2. **Embedded Library**:
   - RAG integrated directly into application code
   - Tight coupling, application-specific customization
   - **Consideration**: Provide integration examples (import patterns, initialization, error handling)

3. **Batch Processing**:
   - Offline analysis, report generation, data enrichment
   - Not interactive query/response
   - **Consideration**: Document batch query patterns; note performance characteristics for bulk operations

4. **Interactive Applications**:
   - Chatbots, Q&A interfaces, documentation assistants
   - Real-time, conversational, user-facing
   - **Consideration**: Provide chat mode example; document conversation history patterns

**Cost Sensitivity**:

Customers have different cost constraints:

1. **API-Based LLMs** (OpenAI, Anthropic):
   - Pro: Easy setup, high quality, no infrastructure
   - Con: Per-token costs, data privacy concerns
   - **Consideration**: Document cost estimation (queries/day × tokens/query × cost/token); show how to set budget limits

2. **Self-Hosted LLMs** (Ollama, vLLM):
   - Pro: No per-query costs, data stays local, unlimited usage
   - Con: Infrastructure costs, GPU requirements, maintenance
   - **Consideration**: Default to self-hosted option (Ollama) for demo; document API-based alternatives

3. **Embedding Model Costs**:
   - Self-hosted (free, but compute cost): sentence-transformers models
   - API-based (cost per embedding): OpenAI embeddings
   - **Consideration**: Demo uses free sentence-transformers; document that embeddings are one-time cost at ingestion

**Security and Privacy**:

Customers have varying security requirements:

1. **Data Sensitivity**:
   - Public information (marketing, documentation)
   - Confidential (internal policies, customer data)
   - Regulated (PII, PHI, financial data)
   - **Consideration**: Document data flow (where documents/embeddings/queries are stored/sent); note that pgvector is local (no external transmission) but LLM choice affects data privacy

2. **Compliance Requirements**:
   - GDPR, HIPAA, SOC2, ISO27001
   - Data residency requirements
   - Audit logging needs
   - **Consideration**: Demo is not compliance-certified; documentation should note that production deployments must add access controls, audit logging, encryption at rest/in transit

3. **Access Control**:
   - Single-tenant (demo scope)
   - Multi-tenant with data isolation
   - Role-based access to documents
   - **Consideration**: Out of scope for demo; document that production implementations need authentication/authorization layers

**Performance Expectations**:

Customers have different performance requirements:

1. **Latency Sensitivity**:
   - Interactive (must respond in <3 seconds for good UX)
   - Asynchronous (batch processing, background jobs)
   - **Consideration**: Document expected query latency; note that first query includes model loading overhead (5-10 seconds)

2. **Throughput Requirements**:
   - Low volume (<100 queries/day): Single container sufficient
   - Medium volume (<10K queries/day): May need multiple replicas
   - High volume (>10K queries/day): Need load balancing, caching, optimization
   - **Consideration**: Demo is single-instance; documentation notes scaling patterns for production

3. **Concurrent Users**:
   - Single user (development/testing)
   - Small team (<10 users)
   - Organization-wide (100s of users)
   - **Consideration**: Demo supports single user; document connection pooling and replica strategies for concurrent access

**Operational Maturity**:

Customers vary in operational capabilities:

1. **Monitoring and Observability**:
   - Basic (logs only)
   - Intermediate (metrics, dashboards)
   - Advanced (distributed tracing, SLOs, alerting)
   - **Consideration**: Demo includes basic logging; document hooks for Prometheus metrics, OpenTelemetry tracing in production

2. **Incident Response**:
   - Manual troubleshooting
   - Runbooks and playbooks
   - Automated recovery
   - **Consideration**: Troubleshooting section provides manual debugging steps; note that production needs automated health checks and restart policies

3. **Change Management**:
   - Ad-hoc updates
   - Scheduled maintenance windows
   - Blue-green deployments, canary releases
   - **Consideration**: Demo is dev-oriented; documentation notes that production updates need zero-downtime strategies (rolling updates, database migration patterns)

**Summary of Key Customer-Driven Requirements**:

- Support resource-constrained environments (minimum 8GB RAM)
- Work in airgapped deployments (pre-loaded models, no external dependencies)
- Provide clear guidance for different document types and sizes
- Accommodate varying technical skill levels with progressive disclosure
- Default to cost-effective self-hosted LLM option (Ollama)
- Document data privacy implications of LLM provider choices
- Set realistic performance expectations for demo vs. production
- Provide clear production adaptation pathway in documentation
- Include troubleshooting for common environment-specific issues
