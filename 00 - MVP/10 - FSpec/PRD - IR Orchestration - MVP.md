# PRD — IR Orchestration

## 1. Executive Summary

Build an information retrieval orchestration layer that sits between the chat interface and the database (which contains prepared keywords and vector embeddings). The orchestration layer receives user queries, retrieves relevant content, assembles context, and passes it to the LLM for generation. The MVP delivers a working internal prototype with four components: session management & API layer, external LLM service integration (via BAML), vector retrieval, and basic context assembly.

## 2. Product Overview

**Purpose:** The IR orchestration layer is the query-to-context pipeline — it takes a user's natural language question, finds the most relevant content from the knowledge base, and assembles it into a prompt the LLM can use to generate grounded answers.

**Key features:**
- REST API layer with session management for FE integration
- External LLM service integration via BAML (embedding + LLM clients)
- Vector-based semantic search against pre-computed embeddings
- Top-k chunk retrieval with relevance-ordered context assembly
- Structured prompt construction with source attribution

**Benefits:**
- Grounds LLM responses in actual organizational knowledge, reducing hallucination
- Provides a modular retrieval layer that can be improved independently of the UI and data pipeline
- Enables internal users to query organizational knowledge through a chat interface

**Strategic alignment:** This is one of three workstreams (alongside data processing and chat UI) that together deliver the RAG application. The orchestration layer is the core intelligence connecting user intent to relevant content.

## 3. Target Personas

**Internal knowledge worker**
- Needs answers grounded in organizational documents, not generic LLM knowledge
- Pain point: information is scattered across documents and hard to find quickly
- Motivation: get accurate, sourced answers through a conversational interface without manual search

## 4. Functional Requirements (Epics)

### Epic 1: Session Management & API Layer

> As a **front-end service**, I want API endpoints to create conversations, send messages, and retrieve history, so that multiple users and windows can interact with the orchestration layer concurrently without cross-contamination.

**In Scope:**
- REST API endpoints for the FE service to call:
  - `POST /sessions` — create a new session, returns a session object containing `session_id`
  - `POST /messages` — FE sends a user query with `session_id` as a required param. Returns the full LLM response via standard HTTP request/response. Accepts an optional `"stream": true` param to enable SSE streaming (future).
  - `GET /sessions/{session_id}` — retrieve the session and all its messages/responses in chronological order
- Generate and assign a unique `session_id` per session
- Store conversation history (user queries + LLM responses) keyed by `session_id` in a persistent database (e.g., PostgreSQL, or the same DB hosting the vector store if it supports relational data)
- Support multiple concurrent conversations across different users/windows (or sessions)
- Session isolation — one conversation's history never leaks into another

**Out of Scope:**
- User authentication / user identity management (FE responsibility)
- Session expiration / TTL (future optimization)
- Conversation search or listing across sessions
- WebSocket / streaming responses (future)

### Epic 2: External LLM Service Integration

> As the **orchestration layer**, I want reliable clients for the embedding API and LLM API, so that the pipeline can convert queries to embeddings and generate responses without each downstream epic managing its own external call logic.

**In Scope:**
- Use BAML as the interface layer between the orchestration layer and the LLM service
- Embedding API client (direct API call, not BAML) — accepts text, returns vector embedding. Used by Vector Retrieval (Epic 3) to convert user queries
- LLM API client (via BAML) — accepts a structured prompt (system + user messages), returns generated text with typed output schemas. Used by Context Assembly (Epic 4) to generate the final response
- Shared error handling: timeouts, rate limit retries (exponential backoff), and graceful degradation on API failure
- API key / config management — externalized configuration, not hardcoded
- Logging of external call latency and errors for debugging

**Out of Scope:**
- Multiple LLM provider support or fallback routing between providers
- Embedding model fine-tuning or selection — use whatever model produced the existing vectors
- Caching of embedding results (future optimization)
- Cost tracking / token usage monitoring

### Epic 3: Vector Retrieval

> As an **internal user**, I want to ask a question and get relevant content retrieved from the knowledge base, so that the LLM can answer based on actual organizational data.

**In Scope:**
- Accept a text query and convert it to a vector embedding via the embedding client (Epic 2)
- Perform similarity search (cosine) against the vector store
- Return top-k most similar chunks (default k=5)
- Return chunks in descending relevance order

**Out of Scope:**
- Keyword/BM25 retrieval (P1.2)
- Hybrid fusion of multiple retrieval paths (P1.2)
- Similarity score threshold filtering (add when irrelevant chunks observed)

### Epic 4: Basic Context Assembly

> As an **internal user**, I want retrieved content assembled into a well-structured prompt, so that the LLM generates accurate, grounded responses with source attribution.

**In Scope:**
- Concatenate top-k chunks into a context block with clear delimiters between chunks
- Include source metadata (document name/identifier) with each chunk
- Number chunks to enable citation in LLM responses (e.g., [1], [2])
- Construct prompt with system instructions separate from context + query
- System prompt instructs LLM to use only provided context and state when information is insufficient
- Send assembled prompt to LLM via the LLM client (Epic 2) and return generated response

**Out of Scope:**
- Dynamic context window budgeting (P1.5)
- Deduplication of overlapping chunks (P1.3)
- Parent-child chunk expansion (P2)
- Chat history inclusion in prompt (P1.1)

### Cross-Cutting Out of Scope

- **Data processing/ingestion pipeline** — chunking, embedding generation, and database writes are a separate workstream
- **Chat UI** — the front-end interface is a separate workstream
- **LLM model selection and tuning** — generation parameters, model comparison, and fine-tuning are outside MVP scope
- **Access control / ACL filtering** — no multi-tenant requirements for internal prototype

## 5. User Stories

### Epic 1: Session Management & API Layer

| Title                    | Story Detail                                                                                                                                                                                                          | Acceptance Criteria                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Priority |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| Create session           | As a **front-end service**, I want to call `POST /sessions` to start a new session, so that each conversation gets its own isolated thread.                                                                           | - Returns a session object: `{ "session_id": string, "created_at": timestamp }`<br>- Session is persisted in the database<br>- Concurrent calls produce distinct IDs                                                                                                                                                                                                                                                                                                                                                                               | High     |
| Send message             | As a **front-end service**, I want to call `POST /messages` with a `session_id` and user query, so that the orchestration layer retrieves context, generates a response, and stores both in the conversation history. | - Request: `{ "session_id": string, "content": string, "stream": boolean (optional, default false) }`<br>- Response: `{ "message_id": string, "session_id": string, "role": "assistant", "content": string, "sources": [{ "index": number, "source": string }], "created_at": timestamp }`<br>- Runs retrieval and context assembly pipeline<br>- Returns full LLM response via standard HTTP<br>- Stores user query and LLM response in conversation history<br>- Returns 403 for invalid `session_id` | High     |
| Retrieve session history | As a **front-end service**, I want to call `GET /sessions/{session_id}`, so that the UI can render the full conversation thread on page load or refresh.                                                              | - Response: session object with `session_id`, `created_at`, and `messages` array. Each message contains `message_id`, `role` (user or assistant), `content`, `sources` (array of index + source, null for user messages), `created_at`<br>- Messages returned in chronological order<br>- Returns empty `messages` array for a new session<br>- Returns 403 for invalid `session_id`                                                                                                                                                               | High     |
| Session isolation        | As a **front-end service**, I want sessions to be fully isolated, so that one user's conversation history never appears in another user's session.                                                                    | - Messages stored under one `session_id` are never returned under a different `session_id`<br>- Concurrent requests to different sessions do not interfere                                                                                                                                                                                                                                                                                                                                                                                         | High     |

**Out of scope for this epic:** User authentication, session expiration/TTL, conversation listing/search, WebSocket streaming.

### Epic 2: External LLM Service Integration

| Title                      | Story Detail                                                                                                                                                                                                              | Acceptance Criteria                                                                                                                                                                                                                                   | Priority |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| BAML integration setup     | As an **engineer**, I want BAML configured as the interface layer for LLM service calls, so that prompt definitions, output schemas, and provider config are managed declaratively.                                       | - BAML is installed and integrated into the project<br>- BAML functions defined for LLM calls<br>- Output types are defined with typed schemas                                                                                                        | High     |
| Embedding API client       | As the **orchestration layer**, I want a direct API client that converts text to vector embeddings, so that Vector Retrieval can query the vector store. (BAML does not handle embeddings — this is a standalone client.) | - Accepts text input, returns vector embedding via direct API call<br>- Uses the same embedding model that produced the stored vectors<br>- Embedding generated within 500ms<br>- Malformed or empty input returns a clear error                      | High     |
| LLM API client             | As the **orchestration layer**, I want a BAML-defined client that sends structured prompts to the LLM and returns typed output, so that Context Assembly can produce grounded answers.                                    | - Accepts system message + user message via BAML function<br>- Returns generated text with typed response schema<br>- Configurable model endpoint via BAML provider config<br>- Returns response within timeout threshold                             | High     |
| Error handling and retries | As the **orchestration layer**, I want external API calls to handle failures gracefully, so that transient errors don't crash the pipeline.                                                                               | - Timeouts configured per service (embedding, LLM)<br>- Rate limit errors trigger exponential backoff retry (max 3 attempts)<br>- On persistent failure, returns a user-friendly error message<br>- All external call errors and latencies are logged | High     |
| API config management      | As an **engineer**, I want API keys and endpoints managed via external configuration, so that credentials are not hardcoded and environments can differ.                                                                  | - API keys loaded from environment variables (stored in env.*)<br>- No credentials in source code<br>- Config supports separate values per environment (dev, staging, prod)                                                                           | High     |

**Out of scope for this epic:** Multiple LLM provider support, embedding caching, cost/token usage monitoring.

### Epic 3: Vector Retrieval

| Title | Story Detail | Acceptance Criteria | Priority |
|---|---|---|---|
| Query embedding conversion | As an **internal user**, I want my text query converted to a vector embedding, so that it can be matched against stored document vectors. | - Query text is passed to the embedding client (Epic 2)<br>- Returns a vector embedding compatible with the stored vectors<br>- Malformed or empty queries return a clear error | High |
| Semantic similarity search | As an **internal user**, I want the system to find the most relevant chunks by cosine similarity, so that I get content that semantically matches my question. | - Cosine similarity search is performed against the vector store<br>- Returns top-k results (default k=5)<br>- Results are sorted in descending similarity order | High |
| Retrieval response structure | As an **internal user**, I want retrieved chunks returned with their similarity scores and source metadata, so that downstream assembly can use relevance ordering and attribution. | - Each result includes: chunk text, similarity score, source metadata<br>- Results are ordered by descending similarity score<br>- Empty result sets (no matches) are handled gracefully | High |

**Out of scope for this epic:** Keyword/BM25 retrieval, hybrid fusion, similarity score threshold filtering.

### Epic 4: Basic Context Assembly

| Title                               | Story Detail                                                                                                                                                                          | Acceptance Criteria                                                                                                                                                                                                                                                                                                                                                          | Priority |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| Chunk concatenation with delimiters | As an **internal user**, I want retrieved chunks assembled into a single context block with clear separators, so that the LLM can distinguish between different source passages.      | - Chunks are concatenated with consistent delimiters (e.g., `---`)<br>- Chunks appear in descending relevance order<br>- Each chunk URL and time stamp is returned as citation                                                                                                                                                                                               | High     |
| Source metadata inclusion           | As an **internal user**, I want each chunk labeled with its source document name, so that I can verify where the information came from.                                               | - Each chunk displays source identifier (document name/filename)<br>- Metadata is formatted consistently across all chunks<br>- Missing metadata is handled gracefully (e.g., "Unknown source")                                                                                                                                                                              | High     |
| Prompt construction                 | As an **internal user**, I want the system to build a structured prompt with system instructions, context, and my query, so that the LLM generates grounded answers.                  | - System instructions are in the system message (separate from context + query)<br>- Context block and user query are in the user message<br>- System prompt instructs LLM to use only provided context<br>- System prompt instructs LLM to state when information is insufficient<br>- Assembled prompt is sent to LLM via the LLM client (Epic 2) and response is returned | High     |
| No-results handling                 | As an **internal user**, I want a clear response when no relevant content is found, so that I know the system doesn't have the information rather than getting a hallucinated answer. | - When retrieval returns zero chunks (or all below future threshold), a "not enough information" response is returned<br>- The LLM is not called with an empty context block                                                                                                                                                                                                 | Medium   |

**Out of scope for this epic:** Dynamic context window budgeting, deduplication, parent-child chunk expansion, chat history inclusion.

## 6. Success Metrics

**Leading indicators:**
- Retrieval latency: p95 < 2 seconds for query-to-context assembly (excludes LLM generation time)
- Top-k relevance: manual spot-check that 3 out of 5 retrieved chunks are relevant to the query (sample of 20 queries)

**Lagging indicators:**
- User satisfaction: >70% of internal testers rate answers as "helpful" or "mostly helpful" after 2 weeks of use
- Hallucination rate: <20% of answers contain claims not supported by retrieved context (measured via manual review of 50 responses)

**[To do]** Refine targets after initial internal testing with real queries.

## 7. Open Questions

| Question | Owner |
|---|---|
| Which embedding model is used for the existing vectors? Need to use the same model for query embedding. | Engineering |
| What vector store/database are the embeddings stored in? Determines the query API. | Engineering |
| What is the average chunk size (tokens) in the current database? Affects k selection and context budget. | Engineering |
| Are there metadata fields stored alongside chunks (source, date, section)? Determines what we can display. | Engineering |
| Which LLM will be used for generation? Affects context window limits and prompt format. | Engineering |
| How many internal users for the prototype? Affects latency/scaling requirements. | Product |

## 8. Roadmap

### Now (MVP)

Minimal viable retrieval pipeline for an internal prototype.

| Feature                         | Description                                                   |
| ------------------------------- | ------------------------------------------------------------- |
| Session Management & API Layer  | REST API endpoints and session persistence for FE integration |
| External LLM Service Integration | Embedding API and LLM API clients via BAML with error handling/retries |
| Dense retrieval (vector search) | Semantic search using existing vector embeddings              |
| Basic context assembly          | Concatenate top-k chunks into LLM prompt in relevance order   |

**Rationale:** These four components are the minimum needed to get a working query-to-answer flow. Session management provides the API surface for FE integration. External service integration provides the plumbing to call embedding and LLM APIs reliably. Everything else improves quality but the system cannot function without these.

### Next (Fast Follow — P1)

Phased rollout to improve retrieval quality. Each phase builds on the previous.

| Phase | Feature | Expected Impact |
|---|---|---|
| P1.1 | Chat history-aware retrieval (last N turns) | Multi-turn coherence |
| P1.2 | BM25 retrieval + hybrid fusion with RRF | 15-30% recall improvement |
| P1.3 | Cross-encoder re-ranking + deduplication | 35% hallucination reduction |
| P1.4 | Query rewriting + coreference resolution | 30-45% precision improvement |
| P1.5 | Context window budgeting + metadata filtering | Operational robustness |

**Rationale:** Ordered by impact to user experience. Chat history comes first because multi-turn is core to a chat interface. Hybrid retrieval and re-ranking deliver the largest measurable quality gains. Query processing and budgeting refine the pipeline once the foundation is solid.

### Future (P2–P3)

See [[IR Orchestration Ideation and Prioritization]] for the full feature list including query decomposition, parent-child retrieval, evaluation frameworks, learned fusion, and advanced conversational features.

**Rationale:** These features optimize beyond baseline quality. Defer until real usage reveals specific failure modes that justify the complexity.

### Boundaries

- **In scope:** Session management & API layer, external LLM service integration (via BAML), vector retrieval, context assembly, and LLM response generation
- **Out of scope:** Data processing/ingestion pipeline, chat UI, LLM model selection/tuning — these are separate workstreams

---

## Appendix

### Architecture

MVP interaction flow showing how the 4 epics connect:

```
FE (Chat UI)
    |
    |  POST /sessions  ──────────►  [Epic 1: Session Management & API Layer]
    |  POST /messages   ──────────►       |
    |  GET /sessions/{id} ────────►       |  Creates session, stores history,
    |                                     |  routes request through pipeline
    |                                     |
    |                                     v
    |                             ┌───────────────────┐
    |                             │  Epic 2: External  │
    |                             │  Service Integration│
    |                             │                     │
    |                             │  ┌───────────────┐ │
    |                             │  │ Embedding API  │ │  Direct API call
    |                             │  │ Client         │◄──── (not BAML)
    |                             │  └───────┬───────┘ │
    |                             │          │         │
    |                             │  ┌───────────────┐ │
    |                             │  │ LLM API Client│ │  Via BAML
    |                             │  │ (BAML)        │◄──── (typed schemas)
    |                             │  └───────┬───────┘ │
    |                             └──────────┼─────────┘
    |                                        │
    |                                        v
    |                             ┌─────────────────────┐
    |                             │  Epic 3: Vector      │
    |                             │  Retrieval            │
    |                             │                       │
    |                             │  1. Query text        │
    |                             │     → Embedding API   │
    |                             │     → vector          │
    |                             │  2. Cosine similarity  │
    |                             │     search             │
    |                             │  3. Return top-k       │
    |                             │     chunks             │
    |                             └───────────┬───────────┘
    |                                         │
    |                                         v
    |                             ┌─────────────────────┐
    |                             │  Epic 4: Basic       │
    |                             │  Context Assembly     │
    |                             │                       │
    |                             │  1. Assemble chunks   │
    |                             │     with delimiters   │
    |                             │     + source metadata  │
    |                             │  2. Build prompt       │
    |                             │     (system + context  │
    |                             │      + query)          │
    |                             │  3. Send to LLM API   │
    |                             │     → get response     │
    |                             └───────────┬───────────┘
    |                                         │
    ◄─────────────────────────────────────────┘
    LLM response returned to FE
    (stored in session history)
```

**Request flow for `POST /messages`:**
1. FE calls `POST /messages` with `session_id` + user query
2. Epic 1 validates `session_id`, stores user message
3. Epic 2 (Embedding client) converts query text → vector embedding
4. Epic 3 runs cosine similarity search → returns top-k chunks
5. Epic 4 assembles chunks into prompt → sends to LLM via Epic 2 (BAML client) → receives generated response
6. Epic 1 stores assistant response in session history, returns to FE
