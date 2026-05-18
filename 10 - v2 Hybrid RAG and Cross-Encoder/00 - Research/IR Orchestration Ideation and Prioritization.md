# IR Orchestration: Ideation and Prioritization

Combined feature list extracted from [[Information Retrieval Orchestration]] and [[PRD - IR Orchestration - MVP]], organized by priority.

**Priority definitions:**
- **P0** — Must have for MVP (cannot function without it)
- **P1** — Must have for "better" performance
- **P2** — Nice to have
- **P3** — Future to do

---

## P0 — MVP (minimum for a working internal prototype)

| Feature | Description | Source |
|---|---|---|
| Session Management & API Layer | REST API endpoints (`POST /sessions`, `POST /messages`, `GET /sessions/{session_id}`) and session persistence for FE integration | Epic 1 |
| External LLM Service Integration | LLM API client via BAML (typed schemas, prompt definitions) + Embedding API client (direct API call). Shared error handling/retries and externalized config | Epic 2 |
| Dense retrieval (vector search) | Semantic search using existing vector embeddings — starting path for MVP | Epic 3 |
| Basic context assembly | Concatenate top-k retrieved chunks into the LLM prompt in relevance order, send to LLM via BAML client | Epic 4 |

## P1 — Better Performance (fast follow)

| Phase | Feature                                         | Description                                                                                    | Source                   |
| ----- | ----------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------ |
| P1.1  | Chat history-aware retrieval (last N turns)     | Include recent turns so follow-up queries retrieve correctly                                   | Conversational Retrieval |
| P1.2  | Sparse / BM25 retrieval                         | Add keyword-based retrieval using prepared keywords                                            | Retrieval Strategies     |
| P1.2  | Hybrid with RRF                                 | Fuse the two retrieval paths with Reciprocal Rank Fusion — 15-30% recall improvement           | Retrieval Strategies     |
| P1.3  | Cross-encoder re-ranking                        | Re-score top-20 results with a cross-encoder for top-5 precision — 35% hallucination reduction | Re-ranking               |
| P1.3  | Deduplication                                   | Remove semantically overlapping chunks — 30-40% of retrieved context is typically redundant    | Context Assembly         |
| P1.4  | Query rewriting                                 | LLM rephrases user input using domain vocabulary — 30-45% precision improvement                | Query Processing         |
| P1.4  | Coreference resolution via query rewriting      | Resolve pronouns/references using chat history before retrieval                                | Conversational Retrieval |
| P1.5  | Context window budgeting                        | Reserve tokens for system prompt, chat history, generation; manage overflow                    | Context Assembly         |
| P1.5  | Basic metadata filtering (date, type, category) | Pre-retrieval filters to narrow search space and improve relevance                             | Metadata Filtering       |

## P2 — Nice to Have

| Feature | Description | Source |
|---|---|---|
| Query decomposition | Breaks complex, multi-part questions into independent sub-queries | Query Processing |
| Lightweight rerankers | Distilled models (FlashRank, zerank-1) for near-cross-encoder quality at lower latency | Re-ranking |
| Parent-child retrieval | Index small chunks for matching, fetch parent chunks for richer LLM context | Context Assembly |
| Access control / ACL filtering | Enforce user/role permissions at the retrieval boundary | Metadata Filtering |
| LLM-extracted filters from natural language | Use LLM function calling to extract structured filter criteria from queries | Metadata Filtering |
| User feedback collection (thumbs up/down) | Capture explicit signals to identify retrieval failures | Evaluation & Feedback |
| Recall@k and NDCG@k measurement | Core retrieval quality metrics for evaluating search results | Evaluation & Feedback |
| End-to-end faithfulness/hallucination evaluation | Measure whether generated answers stay grounded in retrieved context | Evaluation & Feedback |
| LLM-as-judge for scalable evaluation | Use an LLM to evaluate retrieval relevance when human labels are unavailable | Evaluation & Feedback |

## P3 — Future To Do

| Feature | Description | Source |
|---|---|---|
| Query routing | Classify queries to route to the appropriate retrieval strategy or index | Query Processing |
| SPLADE / ColBERT | Learned sparse retrieval or late-interaction models for beyond-baseline optimization | Retrieval Strategies |
| Learned fusion | Train a model to predict optimal combination of retriever scores | Retrieval Strategies |
| LLM-based re-ranking | Prompt an LLM to score/rank passages for high-stakes queries | Re-ranking |
| Hierarchical chunking | Multi-granularity chunk tree (document > section > paragraph > sentence) | Context Assembly |
| Late chunking | Embed full document first, then pool within chunk boundaries for better reference resolution | Context Assembly |
| Semantic memory for long conversations | Store chat history as embeddings, retrieve only relevant turns | Conversational Retrieval |
| Follow-up vs. topic change detection | Detect whether a query continues the current topic or starts a new one | Conversational Retrieval |
| Session state tracking | Track current topic, entities, and user preferences across the session | Conversational Retrieval |
| Graph-based metadata | Use knowledge graph relationships to expand or refine metadata filters | Metadata Filtering |
| Offline test set with CI/CD gating | Gate pipeline changes on metric improvements using test triples | Evaluation & Feedback |
| Feedback-to-training loop | Aggregate user feedback to fine-tune embedding/re-ranking models | Evaluation & Feedback |
| Continuous production monitoring | Track retrieval and generation metrics over time to detect drift | Evaluation & Feedback |
