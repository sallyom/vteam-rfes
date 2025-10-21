# Feature Specification: LlamaIndex + pgvectordb RAG Demo

**Feature Branch**: `001-llamaindex-rag-demo`
**Created**: 2025-10-21
**Status**: Draft
**Input**: User description: "Add an example in demos folder using a docs2db generated pgvectordb dump file to load and run pgvectordb in a pod alongside a llamaindex container. The example should be similar to the Llamaindex & ramalama example. Include the steps to take a folder of documents and create the pgvectordb dump file. Provide a clear and concise end-to-end README.md to document how to run the example."

## User Scenarios & Testing

### User Story 1 - First-Time RAG Setup with LlamaIndex (Priority: P1)

A developer or data scientist wants to quickly set up a working RAG system using their own documents. They have documents they want to query and need to go from raw files to a functioning question-answering system with minimal friction.

**Why this priority**: This is the core value proposition - enabling users to go from documents to a queryable system. Without this working, no other functionality matters.

**Independent Test**: Can be fully tested by running the complete workflow (ingest documents → create dump → deploy containers → execute query) and verifying that queries return relevant answers with source citations. Delivers immediate value by providing a functional RAG system.

**Acceptance Scenarios**:

1. **Given** a folder containing 10-20 PDF documents, **When** the user follows the README instructions to ingest documents and create a database dump, **Then** the dump file is created successfully with all document chunks embedded
2. **Given** a created database dump file, **When** the user deploys the pod using the provided pod.yaml, **Then** both pgvector and LlamaIndex containers start successfully within 5 minutes
3. **Given** running containers, **When** the user executes a query like "What is pgvector?", **Then** the system returns a relevant answer with at least 3 source document citations in under 5 seconds
4. **Given** a successful first query, **When** the user runs multiple follow-up queries, **Then** each query completes successfully with relevant results

---

### User Story 2 - Understanding RAG Pipeline Mechanics (Priority: P2)

A developer learning about RAG wants to understand how retrieval-augmented generation works under the hood. They need visibility into each step of the pipeline to debug issues and make informed customization decisions.

**Why this priority**: Differentiates this demo from black-box solutions. Users can learn and troubleshoot effectively, which increases adoption and reduces support burden.

**Independent Test**: Can be tested by running queries with the explain mode and verifying that all pipeline steps (embedding generation, vector search, context assembly, LLM prompting) are clearly displayed with intermediate values.

**Acceptance Scenarios**:

1. **Given** a running RAG system, **When** the user runs a query with explain flag, **Then** the system displays embedding generation details, similarity scores for retrieved chunks, and the prompt sent to the LLM
2. **Given** query results with low relevance, **When** the user examines the explanation output, **Then** they can identify whether the issue is in retrieval (poor similarity scores) or generation (irrelevant synthesis)
3. **Given** the architecture documentation, **When** a user reviews the data flow diagram and component descriptions, **Then** they understand how documents flow from ingestion through query response

---

### User Story 3 - Customizing LLM Provider (Priority: P3)

A user wants to switch between different LLM providers (local Ollama models, OpenAI, Anthropic) to compare response quality or adapt to their infrastructure constraints.

**Why this priority**: Provides flexibility and demonstrates framework-agnostic approach. Not critical for initial functionality but important for production adaptation.

**Independent Test**: Can be tested by modifying the configuration file to switch LLM providers and verifying that queries work with each provider without code changes.

**Acceptance Scenarios**:

1. **Given** a running system with default Ollama configuration, **When** the user edits config.yaml to specify OpenAI and sets the API key, **Then** queries use OpenAI GPT-4 for generation after container restart
2. **Given** multiple LLM provider configurations, **When** the user compares query results across providers, **Then** they receive answers from each provider while maintaining consistent retrieval
3. **Given** an airgapped environment with no external API access, **When** the user configures a local Ollama model, **Then** the system operates entirely without external network calls

---

### User Story 4 - Interactive Query Exploration (Priority: P3)

A user wants to have a conversational interaction with their document corpus, asking follow-up questions and exploring topics interactively rather than running one-off queries.

**Why this priority**: Enhances user experience but is not essential for demonstrating core RAG functionality. Can be added after basic query capability is proven.

**Independent Test**: Can be tested by launching the interactive chat mode and verifying that conversation history is maintained across multiple related queries.

**Acceptance Scenarios**:

1. **Given** a running RAG system, **When** the user starts interactive chat mode, **Then** they can enter multiple queries in sequence with conversation context maintained
2. **Given** an active chat session, **When** the user asks a follow-up question referencing previous context, **Then** the system understands the reference and provides relevant answers
3. **Given** chat mode, **When** the user types special commands like /explain or /sources, **Then** the system displays detailed information about the last query

---

### Edge Cases

- What happens when the database dump file is missing or corrupted during container initialization?
- How does the system handle embedding model mismatches between ingestion time and query time?
- What occurs when a query returns no relevant results (no documents match above similarity threshold)?
- How does the system behave when the LLM service is unavailable or returns errors?
- What happens when multiple users try to query simultaneously with a single-instance deployment?
- How does the system handle very long documents that exceed context window limits?
- What occurs when the pgvector database connection drops during query execution?
- How does the system respond to malformed queries or queries in unsupported languages?

## Requirements

### Functional Requirements

- **FR-001**: Demo MUST provide containerized deployment using pod.yaml with two containers: pgvector database and LlamaIndex service
- **FR-002**: pgvector container MUST load a docs2db-generated database dump file (ragdb_dump.sql) on initialization using a custom entrypoint script
- **FR-003**: LlamaIndex service MUST connect to pgvector database and successfully initialize a vector store from the loaded database
- **FR-004**: Demo MUST support query execution via Python CLI that retrieves relevant document chunks and synthesizes answers using an LLM
- **FR-005**: Documentation MUST show complete workflow: documents folder → docs2db ingestion → database dump creation → container deployment → query execution
- **FR-006**: Demo MUST verify embedding model compatibility - LlamaIndex MUST use the same embedding model as docs2db ingestion (default: all-MiniLM-L6-v2)
- **FR-007**: README.md MUST include prerequisites, build steps, deployment steps, verification steps, and troubleshooting guidance
- **FR-008**: Demo MUST provide example queries that demonstrate successful RAG functionality with source attribution and relevance scores
- **FR-009**: Demo MUST handle schema compatibility between docs2db pgvector table structure and LlamaIndex PGVectorStore expectations
- **FR-010**: Documentation MUST specify resource requirements (RAM, CPU, disk space) with tested minimum and recommended configurations
- **FR-011**: Demo MUST provide configuration file (config.yaml) for tuning retrieval parameters (chunk size, top-k results, similarity threshold) and LLM provider selection
- **FR-012**: Demo MUST support multiple LLM backends (Ollama, OpenAI, Anthropic) with clear provider switching instructions
- **FR-013**: Demo MUST include health check mechanisms to verify database readiness and service availability
- **FR-014**: Demo MUST provide sample documents or clear instructions for using custom document sets
- **FR-015**: Demo MUST implement graceful error handling with clear error messages and remediation steps
- **FR-016**: Documentation MUST include architecture diagram showing data flow from documents through ingestion to query response
- **FR-017**: Demo MUST provide comparison context explaining when to use LlamaIndex demo vs. Llama Stack demo with decision criteria
- **FR-018**: Demo MUST include verification commands that users can run to confirm successful setup at each stage
- **FR-019**: Demo MUST support idempotent database initialization - detecting existing database and skipping reload on restart
- **FR-020**: Demo MUST document expected performance characteristics (query latency, memory usage, disk space requirements) based on tested configurations

### Key Entities

- **Document**: Original files (PDF, Markdown, text) that users want to query. Key attributes include file path, content type, size, and metadata
- **Chunk**: Segmented portion of a document created during ingestion. Attributes include text content, source document reference, position in document, and metadata
- **Embedding**: Vector representation of a chunk's semantic meaning. Attributes include dimension (384 for all-MiniLM-L6-v2), embedding model identifier, and generation timestamp
- **Query**: User's natural language question. Attributes include query text, embedding, retrieval parameters (top-k, similarity threshold)
- **Retrieved Context**: Set of chunks selected as relevant to a query. Attributes include chunk text, similarity scores, source document references
- **Answer**: LLM-generated response synthesized from retrieved context. Attributes include answer text, source citations, confidence indicators

## Success Criteria

### Measurable Outcomes

- **SC-001**: Users can complete the entire workflow from raw documents to first successful query in under 15 minutes (excluding model download time)
- **SC-002**: Query response time is under 5 seconds for typical queries after initial model loading
- **SC-003**: Database dump load completes within 5 minutes for typical document sets (1,000-5,000 chunks, ~500MB dump file)
- **SC-004**: Total pod memory usage remains under 8GB with default configuration running locally
- **SC-005**: Container deployment succeeds on first attempt when following README instructions for users with prerequisites met
- **SC-006**: 90% of queries return at least 3 relevant source document chunks with similarity scores above 0.5
- **SC-007**: Documentation walkthrough can be completed successfully by a developer not involved in implementation within 30 minutes
- **SC-008**: All commands in README work without modification when copy-pasted in a fresh environment
- **SC-009**: Users can switch LLM providers by editing configuration and restarting containers without code changes
- **SC-010**: System handles at least 20 consecutive queries without degradation or memory leaks
