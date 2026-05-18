---
status: Accepted
created_at: 2026-04-09
accepted_at: 2026-04-18
source:
  - "[[Retrieval Quality Baseline and Evaluation Strategy]]"
  - "[[IR Orchestration Ideation and Prioritization]]"
  - "[[PRD - IR Orchestration - MVP]]"
Stage: Information Retrieval Orchestration
---

# PRD — IR Orchestration: Orchestration Quality Tuning v1

## 1. Executive Summary

Upgrade the MVP's naive dense retrieval (vector-only, top-k cosine) to a hybrid retrieval pipeline with keyword search, rank fusion, and re-ranking. Before building, instrument the current pipeline with logging and establish a measurable baseline so that each improvement can be validated against data, not gut feel. This fast follow covers three areas: evaluation infrastructure (logging + test set + baseline), hybrid retrieval (BM25 + RRF), and result refinement (cross-encoder re-ranking + deduplication).

## 2. Product Overview

**Purpose:** The MVP retrieves chunks via dense vector search only. This works for queries that are semantically close to the stored content, but misses cases where exact keyword matches matter (e.g., product names, policy numbers, acronyms). Adding keyword-based retrieval and fusing it with dense retrieval broadens coverage. Re-ranking then sharpens precision by rescoring the fused results with a cross-encoder. Deduplication removes redundant chunks so the LLM's context budget isn't wasted on repeated information.

**Key features:**
- Retrieval pipeline logging at both retrieval and generation stages
- Evaluation test set and baseline measurement infrastructure
- BM25 keyword-based retrieval using prepared keywords
- Hybrid fusion via Reciprocal Rank Fusion (RRF)
- Cross-encoder re-ranking of fused results
- Semantic deduplication of overlapping chunks

**Benefits:**
- Expected 15–30% recall improvement from hybrid retrieval (industry benchmarks)
- Expected 35% hallucination reduction from re-ranking (cross-encoder precision gains)
- 30–40% context waste reduction from deduplication
- Data-driven evaluation framework to validate each improvement and guide future work

**Strategic alignment:** This is the first quality-improvement iteration on the retrieval pipeline. The evaluation infrastructure built here will be reused for all future P1 phases (query rewriting, chat history, context budgeting).

## 3. Target Personas

**Engineering team**
- Needs to measure whether pipeline changes actually improve retrieval quality
- Pain point: no way to quantify current performance or validate improvements — changes are evaluated by spot-checking, not data
- Pain point: dense-only retrieval misses keyword-exact matches (vocabulary mismatch), but no metrics to prove it or measure the fix
- Motivation: data-driven development cycle — instrument, measure baseline, change, re-measure, ship or revert

## 4. Functional Requirements (Epics)

### Epic 1: Retrieval Pipeline Logging

> As an **engineer**, I want the retrieval and generation pipeline to log structured data at each stage, so that I can compute evaluation metrics offline without re-running queries.

**In Scope:**
- Log at retrieval time: query text, k value, ordered top-k chunk IDs, per-chunk similarity score
  - With pgvector on Postgres, the similarity score is returned by the DB — the SQL query itself computes the distance/similarity. The service layer receives it as part of the query result. Log it as-is from the query response.
- Log at LLM response time: LLM relevance confidence (`high` / `mid` / `low`), full LLM response text, sources cited by the LLM
  - Extend the BAML function schema to include a `relevance_confidence` field (`high` / `mid` / `low`) in the typed response output. The LLM classifies its own confidence as part of generation — no separate call needed.
- Store logs in a dedicated `retrieval_orchestration_logs` table, separate from the `messages` table
  - Keep `messages` clean for end-user API interactions. A new `retrieval_orchestration_logs` table stores the internal pipeline data (retrieval inputs/outputs, similarity scores, LLM confidence). Linked to `messages` via `message_id` foreign key so you can join when needed for analysis.
  - Include a two-tiered enum for each log entry: `pipeline_stage` (where in the pipeline) and `retrieval_method` (which implementation). See appendix for full enum definitions. Stages for this PRD: `dense_retrieval`, `sparse_retrieval`, `fusion`, `reranking`, `deduplication`, `similarity_floor`, `generation`. The stage enum is the stable contract; the method enum captures tactical implementation details.
- Logging must add < 50ms total across all log writes per request (combined retrieval + generation stage inserts). For reference, the MVP retrieval latency target is p95 < 2 seconds — logging should be < 2.5% of that budget.

**Out of Scope:**
- Real-time dashboards or alerting on logged data (future)
- Log retention policies or rotation
- Logging of chat history or session-level aggregations (P1.1 scope)

### Epic 2: LLM-Derived Baseline (label-free)

> As an **engineer**, I want a baseline of retrieval quality derived from LLM response signals and similarity scores, so that I can measure improvements without needing a labeled test set first.

**In Scope:**
- Evaluation script that computes metrics from Epic 1 logs — no labeled data required
- Baseline metrics from LLM signals:
	- Low-confidence rate (`low` count / total queries)
	- Confidence distribution (`high` / `mid` / `low`)
	- Similarity score → confidence correlation: plot per-chunk similarity scores against the LLM's confidence level for that query (e.g., "when avg similarity drops below 0.5, LLM returns `low` confidence 80% of the time"). This reveals the relationship between retrieval quality and LLM output quality.
- Similarity score analysis:
	- Distribution of per-chunk scores
	- Empirical breakpoint where low-confidence rate spikes
- Stored baseline snapshot for comparison after each pipeline change
- Re-run after each retrieval change (P1.2, P1.3) and compare to previous snapshot — **user-initiated**, not automated. Engineer reviews results and confirms the baseline is valid before it becomes the reference point.
- BM25-specific diagnostics (run after Epic 4 is deployed, using logged traces):
	- **Term hit rate:** fraction of queries where at least one query term matches a retrieved chunk's prepared keywords
	- **Empty result rate:** fraction of queries where BM25 returns zero results
	- **Dense vs. BM25 overlap:** Jaccard similarity between BM25 top-k and dense top-k per query. Low overlap = high value from hybrid fusion; high overlap = BM25 isn't adding much.

**Out of Scope:**
- Labeled test set or ground truth chunk labels (Epic 3)
- Standard IR metrics (Recall@k, Precision@k, MRR) — require labels (Epic 3)
- Automated CI/CD gating on metric thresholds (P2)

### Epic 3: Labeled Test Set and Standard IR Metrics

> As an **engineer**, I want a labeled test set with ground truth relevant chunks, so that I can compute Recall@k, Precision@k, and MRR for rigorous retrieval evaluation.

**In Scope:**
- Test set of query → relevant chunk pairs (minimum 20–30 queries to start)
- Use existing synthetic dataset generation service (separate project: SGDSG) to create the baseline test set
- Validate IR metrics against the synthetic test set to identify retrieval gaps
- Binary relevance labels to start (relevant / not relevant per chunk)
- Evaluation script that computes Recall@k, Precision@k, MRR against the test set
- Baseline snapshot using the labeled set for comparison after each pipeline change
- Can be built in parallel with Epic 2 or after — not a blocker for retrieval changes

**Out of Scope:**
- NDCG@k — requires graded labels
- LLM-as-judge evaluation (P2)

### Epic 4: BM25 Retrieval + Hybrid Fusion (P1.2)

> As an **engineer**, I want keyword-based retrieval combined with semantic search, so that queries with exact terminology (product names, policy numbers, acronyms) find the right content even when embeddings miss.

**In Scope:**
- BM25 keyword-based retrieval using prepared keywords already stored in the database
- Widen retrieval top-k for both dense and BM25 — MVP hardcoded k=5, which is too narrow for a hybrid pipeline that feeds into re-ranking. Make k **configurable per retriever** and empirically tune using Epic 2 and Epic 3 evaluation runs at multiple k values (e.g., k=10, 20, 30). Pick the k that balances recall gains against cross-encoder latency (more candidates = higher recall but slower re-ranking). The final chunk count reaching the LLM is controlled by Epic 5's similarity floor, not by retrieval k. Note: the retrieval-stage floor (below) makes the *effective* candidate count dynamic per query even with a fixed k — if k=20 but the floor drops 8 low-scoring chunks, only 12 enter RRF.
- Retrieval-stage similarity floor: apply a **lenient** threshold on each retriever's results before fusion to drop clearly irrelevant chunks. This reduces noise entering RRF, lowers cross-encoder latency (fewer pairs to re-score), and avoids diluting the fused list. Must be more lenient than Epic 5's post-reranking floor — the goal is to remove obvious garbage, not to be selective.
- Reciprocal Rank Fusion (RRF) to combine dense and BM25 result sets into a single ranked list
- RRF weighting parameter (default equal weight, configurable)
- Fused result set replaces the dense-only result set in the pipeline — context assembly (MVP PRD, Epic 4) consumes the fused results

**Out of Scope:**
- Full-text indexing or re-indexing of source documents (use prepared keywords as-is)
- Learned fusion or trainable score combination (P3)
- Replacing BM25 with more advanced sparse or late-interaction retrieval models (e.g., SPLADE, ColBERT) — P3, only if hybrid BM25 + dense proves insufficient
- Query expansion or synonym handling (P1.4 — query rewriting)

### Epic 5: Cross-Encoder Re-Ranking + Deduplication (P1.3)

> As an **engineer**, I want the top retrieved results re-scored for precision and duplicates removed, so that the LLM receives the most relevant, non-redundant context possible.

**In Scope:**
- Retrieve a wider candidate set from hybrid fusion (size determined by Epic 4's configurable k and retrieval-stage floor)
- Re-score candidates using a cross-encoder model (query + chunk pair scoring)
- After re-ranking, apply a **similarity floor** — only pass chunks that score above a threshold to context assembly, rather than a fixed top-5. This means the number of chunks sent to the LLM varies per query (could be 2, could be 8).
- Semantic deduplication: detect and remove chunks with high overlap before passing to context assembly
- Re-ranking via **Vertex AI Ranking API** — aligns with existing GCP stack (Gemini LLM, Google embedding model). Validate against Epic 3 test set to confirm Precision@5 gains. Must re-rank 20 candidates in < 500ms total.
- Deduplication threshold calibration: run deduplication at varying thresholds (e.g., 0.85, 0.90, 0.95) on a sample of retrieved results, manually inspect which chunks get merged vs. kept, pick the threshold that removes clear duplicates without collapsing distinct-but-related content
- Similarity floor: ship with a **configurable static threshold** and a sensible default. Empirical calibration from live post-reranking data is deferred — Epic 2's baseline data is pre-reranking and won't reflect the new score distribution. Once Epic 5 is live with enough queries logged, re-run Epic 2's similarity score → confidence analysis on the new data to tune the floor.

**Out of Scope:**
- Lightweight/distilled rerankers as a latency optimization (P2)
- LLM-based re-ranking (P3)
- Parent-child chunk expansion (P2)

### Cross-Cutting Out of Scope

- **Chat history-aware retrieval** (P1.1) — separate fast follow phase
- **Query rewriting and coreference resolution** (P1.4) — depends on P1.1
- **Context window budgeting** (P1.5) — operational robustness, not retrieval quality
- **Metadata filtering** (P1.5) — useful but not required to validate hybrid + re-ranking
- **Data processing / ingestion pipeline changes** — separate workstream
- **Chat UI changes** — no UI impact from these backend improvements

## 5. User Stories

### Epic 1: Retrieval Pipeline Logging

| Title | Story Detail | Acceptance Criteria | Priority |
|---|---|---|---|
| Retrieval-time logging | As an **engineer**, I want each retrieval call to log the query, k value, chunk IDs, and similarity scores to `retrieval_orchestration_logs`, so that I can reconstruct what was retrieved for any query. | - Every retrieval call logs: query text, k, ordered chunk IDs, per-chunk similarity score (native from pgvector)<br>- Each log entry includes `pipeline_stage` and `retrieval_method` enum values<br>- Log entries linked to corresponding `message_id` via FK<br>- Logging adds < 50ms total per request across all stages | High |
| Generation-time logging | As an **engineer**, I want each LLM response to log the relevance confidence and sources cited to `retrieval_orchestration_logs`, so that I can measure confidence distribution and chunk utilization. | - Every LLM response logs: `relevance_confidence` (`high` / `mid` / `low`), full response text, cited source chunk IDs<br>- `relevance_confidence` field added to BAML function schema typed output<br>- Confidence classified by the LLM as part of generation — no separate call<br>- Log entry appended to same `message_id` row as retrieval log | High |
| Log table schema | As an **engineer**, I want `retrieval_orchestration_logs` as a dedicated table separate from `messages`, so that internal pipeline data doesn't pollute the end-user API surface. | - New `retrieval_orchestration_logs` table created with `message_id` FK to `messages`<br>- Two-tiered enum columns: `pipeline_stage` (7 values) and `retrieval_method`<br>- JSON column for stage-specific payload (chunk IDs, scores, etc.)<br>- `messages` table unchanged | High |

**Out of scope for this epic:** Real-time dashboards or alerting, log retention policies, chat history or session-level log aggregations (P1.1).

### Epic 2: LLM-Derived Baseline (label-free)

| Title | Story Detail | Acceptance Criteria | Priority |
|---|---|---|---|
| Confidence distribution report | As an **engineer**, I want to compute the distribution of LLM relevance confidence (`high` / `mid` / `low`) across all logged queries, so that I have a label-free measure of how well retrieval is performing. | - Script reads from `retrieval_orchestration_logs`<br>- Outputs: total queries, count and percentage for each confidence level<br>- Low-confidence rate (`low` / total) is the headline metric | High |
| Similarity score → confidence correlation | As an **engineer**, I want to see the relationship between per-chunk similarity scores and LLM confidence levels, so that I can identify the score range where retrieval quality degrades. | - Script computes avg/min/max similarity scores per query, grouped by LLM confidence level<br>- Outputs a mapping (e.g., "avg similarity < 0.5 → `low` confidence 80% of the time")<br>- Identifies the empirical breakpoint where low-confidence rate spikes | High |
| Baseline snapshot storage | As an **engineer**, I want baseline results stored as a versioned snapshot, so that I can compare against it after pipeline changes. | - Snapshot includes: date, pipeline config, per-query results, aggregate metrics<br>- Stored in a portable format (JSON)<br>- Timestamped or versioned for comparison | High |
| Baseline re-run and comparison | As an **engineer**, I want to re-run the evaluation after a pipeline change and compare to the previous snapshot, so that I can confirm improvement or detect regression. | - Re-run is **user-initiated** (not automated)<br>- Outputs side-by-side comparison: previous snapshot vs. new results<br>- Engineer reviews and confirms before the new snapshot becomes the reference point | High |
| BM25 diagnostics | As an **engineer**, I want label-free diagnostics on BM25 retrieval using logged traces, so that I can assess whether keyword search is adding value to the hybrid pipeline. | - **Term hit rate:** fraction of queries where at least one query term matches a retrieved chunk's prepared keywords<br>- **Empty result rate:** fraction of queries where BM25 returns zero results<br>- **Dense vs. BM25 overlap:** Jaccard similarity between BM25 top-k and dense top-k per query (low overlap = high value from fusion)<br>- Runs after Epic 4 is deployed, using traces from `retrieval_orchestration_logs` | High |

**Out of scope for this epic:** Labeled test set or ground truth chunk labels (Epic 3), standard IR metrics (Recall@k, Precision@k, MRR), automated CI/CD gating on metric thresholds (P2).

### Epic 3: Labeled Test Set and Standard IR Metrics

| Title                         | Story Detail                                                                                                                                                                                                     | Acceptance Criteria                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Priority |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| Synthetic test set generation | As an **engineer**, I want a labeled test set of query → relevant chunk pairs generated from existing content, so that I have ground truth to compute IR metrics against.                                        | - Minimum 20–30 queries with labeled relevant chunks<br>- Generated using these two corpus as references:  <br>    — [https://github.com/ai-that-works/ai-that-works/blob/main/2026-01-13-applying-12-factor-principles-to-coding-agent-sdks/transcript.md](https://github.com/ai-that-works/ai-that-works/blob/main/2026-01-13-applying-12-factor-principles-to-coding-agent-sdks/transcript.md)  <br>    — [https://github.com/ai-that-works/ai-that-works/blob/main/2026-02-10-agentic-backpressure-deep-dive/transcript.txt](https://github.com/ai-that-works/ai-that-works/blob/main/2026-02-10-agentic-backpressure-deep-dive/transcript.txt)\|<br>- Binary relevance labels (relevant / not relevant per chunk)<br>- Stored in a portable format (JSON) | High     |
| IR metrics evaluation script  | As an **engineer**, I want a script that runs the test set through the retrieval pipeline and computes Recall@k, Precision@k, and MRR, so that I can rigorously measure retrieval quality.                       | - Script accepts: test set file + retrieval pipeline endpoint or function<br>- Outputs: Recall@k, Precision@k, MRR per query and averaged<br>- Results stored as a snapshot file for comparison                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | High     |
| Labeled baseline run          | As an **engineer**, I want to run the evaluation script against the current dense-only pipeline and store the results, so that I have a labeled baseline snapshot to compare against after each pipeline change. | - Evaluation script run against current dense-only retrieval<br>- Recall@k, Precision@k, MRR computed and stored as the labeled baseline snapshot<br>- Snapshot includes: date, pipeline config, per-query results, aggregate metrics<br>- Run is **user-initiated** — engineer reviews results before the snapshot becomes the reference point                                                                                                                                                                                                                                                                                                                                                                                                                | High     |
| Gap identification            | As an **engineer**, I want to validate IR metrics against the test set to identify specific retrieval gaps, so that I know which query types or content areas the pipeline struggles with.                       | - Script flags queries with Recall@k = 0 (no relevant chunks retrieved)<br>- Groups low-performing queries by pattern (e.g., keyword-heavy, acronym, multi-topic)<br>- Output informs which retrieval improvements to prioritize                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Medium   |

**Out of scope for this epic:** Graded relevance labels (0–3 scale), NDCG@k, LLM-as-judge evaluation (P2).

### Epic 4: BM25 Retrieval + Hybrid Fusion (P1.2)

| Title                            | Story Detail                                                                                                                                                                        | Acceptance Criteria                                                                                                                                                                                                                                                                                                             | Priority |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| Widen retrieval top-k            | As an **engineer**, I want to make retrieval k configurable per retriever and empirically tune it, so that RRF and re-ranking have an optimal candidate set size.                   | - k is configurable per retriever (dense and BM25 independently)<br>- Evaluate at multiple k values (e.g., 10, 20, 30) using Epic 2 and Epic 3 metrics<br>- Pick k that balances recall gains against cross-encoder latency<br>- Final chunk count reaching the LLM is controlled by Epic 5's similarity floor, not retrieval k | High     |
| Retrieval-stage similarity floor | As an **engineer**, I want a lenient similarity threshold applied to each retriever's results before fusion, so that clearly irrelevant chunks don't enter RRF.                     | - Threshold applied independently to dense and BM25 results after retrieval, before RRF<br>- More lenient than Epic 5's post-reranking floor — drops obvious garbage only<br>- Configurable per retriever<br>- Reduces candidate count entering RRF and cross-encoder (lower latency)                                           | High     |
| BM25 keyword retrieval           | As an **engineer**, I want keyword-based search against prepared keywords in the database, so that exact-match queries find the right content.                                      | - BM25 search runs against prepared keywords in Postgres<br>- Full-text search index (e.g., `tsvector`/`tsquery`) exists or is created as part of implementation<br>- Returns top-k results ranked by BM25 score<br>- Handles queries with no keyword matches gracefully (returns empty set)<br>- Logs to `retrieval_orchestration_logs` with stage `sparse_retrieval`, method `bm25`                                                   | High     |
| Reciprocal Rank Fusion           | As an **engineer**, I want dense and BM25 results fused into a single ranked list using RRF, so that the pipeline benefits from both retrieval strategies.                          | - RRF combines dense and BM25 result sets using reciprocal rank formula<br>- Default: equal weighting between dense and BM25<br>- Weighting parameter is configurable<br>- Fused list replaces dense-only results in the pipeline<br>- Logs to `retrieval_orchestration_logs` with stage `fusion`, method `rrf`                 | High     |
| Hybrid retrieval validation      | As an **engineer**, I want to re-run both evaluation tracks (Epic 2 and Epic 3) against hybrid retrieval and compare to the dense-only baseline, so that I can confirm improvement. | - Same test set and log-based metrics used as baseline<br>- Recall@k, Precision@k, MRR compared to baseline snapshot<br>- Low-confidence rate compared to baseline<br>- No regression on any metric<br>- Results stored as a new snapshot                                                                                       | High     |

**Out of scope for this epic:** Full-text indexing or re-indexing of source documents, learned fusion or trainable score combination (P3), replacing BM25 with advanced sparse/late-interaction models (P3), query expansion or synonym handling (P1.4).

### Epic 5: Cross-Encoder Re-Ranking + Deduplication (P1.3)

| Title                      | Story Detail                                                                                                                                                                               | Acceptance Criteria                                                                                                                                                                                                                                                                                                                                              | Priority |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| Re-ranking via Vertex AI   | As an **engineer**, I want the fused candidate set re-scored using the Vertex AI Ranking API, so that the most relevant chunks are prioritized for the LLM.                                | - Re-score each (query, chunk) pair in the fused candidate set via Vertex AI Ranking API<br>- Re-ranking latency scales with candidate count — must stay within acceptable bounds<br>- Logs to `retrieval_orchestration_logs` with stage `reranking`, method `cross_encoder`                                                                                     | High     |
| Similarity floor filtering | As an **engineer**, I want only chunks above a configurable similarity threshold passed to context assembly, so that the LLM receives a dynamic number of high-relevance chunks per query. | - Chunks below the threshold filtered out after re-ranking<br>- Threshold is configurable with a sensible static default<br>- Number of chunks varies per query (could be 2, could be 8)<br>- Logs to `retrieval_orchestration_logs` with stage `similarity_floor`                                                                                               | High     |
| Semantic deduplication     | As an **engineer**, I want overlapping chunks removed after re-ranking, so that the LLM context isn't wasted on redundant information.                                                     | - Chunks with similarity above deduplication threshold are merged or dropped<br>- Deduplication runs after re-ranking, before context assembly<br>- At least one chunk from each duplicate cluster is retained<br>- Threshold calibrated by testing at 0.85, 0.90, 0.95 on sample results<br>- Logs to `retrieval_orchestration_logs` with stage `deduplication` | High     |
| Re-ranking validation      | As an **engineer**, I want to re-run both evaluation tracks against the re-ranked hybrid pipeline and compare to the hybrid-only snapshot, so that I can confirm improvement.              | - Same test set and log-based metrics used as hybrid baseline<br>- Recall@k, Precision@k, MRR compared to hybrid snapshot<br>- Low-confidence rate compared to hybrid snapshot<br>- No regression on any metric<br>- Results stored as a new snapshot                                                                                                            | High     |

**Out of scope for this epic:** Lightweight/distilled rerankers as a latency optimization (P2), LLM-based re-ranking (P3), parent-child chunk expansion (P2).

## 6. Success Metrics

**Leading indicators:**

| Metric | Baseline (dense-only) | Target after P1.2 (hybrid) | Target after P1.3 (re-ranking) |
|---|---|---|---|
| Recall@5 | [To do] Measure | ≥ 10% relative improvement | ≥ 15% relative improvement over baseline |
| MRR | [To do] Measure | No regression | Improvement over hybrid |
| Low-confidence rate | [To do] Measure | Lower than baseline | Lower than hybrid |
| Retrieval latency (p95) | [To do] Measure | < 2x baseline latency | < 3x baseline latency |

**Lagging indicators:**
- User satisfaction: improvement over MVP baseline in "helpful" / "mostly helpful" ratings (measured via same manual review process as MVP)
- Hallucination rate: reduction from MVP baseline (measured via manual review of 50 responses, same methodology)

[To do] Fill in baseline values after Epic 2 is complete. Adjust targets if baseline is already high.

## 7. Open Questions

| Question | Owner | Notes |
|---|---|---|
| What similarity metric does the vector store return natively? | Engineering | Cosine similarity vs. Euclidean/L2 — log whichever it returns, don't convert |
| Are the prepared keywords in the DB suitable for BM25 as-is? | Engineering | Check format, coverage, whether a BM25 index exists or needs creation |
| Which cross-encoder model? | Engineering | Tradeoff: quality vs. latency. Candidates: ms-marco-MiniLM, BGE-reranker, Cohere rerank API |
| What deduplication similarity threshold? | Engineering | Need to experiment — too aggressive loses distinct-but-related chunks, too lenient keeps redundancy |
| How many real user queries from MVP usage? | Engineering | Determines test set bootstrap strategy (synthetic vs. real vs. hybrid) |
| Who labels relevance for the test set? | Engineering / Product | Engineer judgment vs. domain expert — affects quality and speed |
| Where to store test sets and evaluation snapshots? | Engineering | JSON in repo, DB table, or lightweight eval tool |
| Latency budget for the full pipeline? | Engineering / Product | Adding BM25 + re-ranking adds compute — what's the acceptable ceiling? |
| What k values to evaluate at? | Engineering | MVP default is k=5. Evaluate at multiple values (e.g., 10, 20, 30) per retriever using Epic 2 and Epic 3 metrics. Balance recall gains against cross-encoder latency. |

## 8. Roadmap

### Now

Logging and label-free baseline — minimum instrumentation before changing anything.

| Feature | Epic | Description |
|---|---|---|
| Retrieval pipeline logging | 1 | Structured logging at retrieval and generation stages |
| LLM-derived baseline | 2 | Refusal rate, confidence rate, similarity score → outcome mapping from logs |

**Rationale:** Label-free evaluation gets us a baseline fast using data we're already generating. No test set creation needed — just instrument and measure.

### Next

Labeled evaluation + hybrid retrieval.

| Feature | Epic | Description |
|---|---|---|
| Labeled test set + IR metrics | 3 | Query → chunk pairs with Recall@k, Precision@k, MRR |
| BM25 retrieval | 4 | Keyword-based search using prepared keywords |
| RRF fusion | 4 | Combine dense + BM25 into a single ranked list |
| Validation | 2, 3 | Re-run both evaluation tracks, compare to baseline |

**Rationale:** The labeled test set can be built in parallel with hybrid retrieval work. BM25 + RRF addresses the biggest gap in dense-only retrieval (vocabulary mismatch, exact-term queries). Expected 15–30% recall improvement.

### After

Result refinement — precision gains on top of hybrid retrieval.

| Feature | Epic | Description |
|---|---|---|
| Cross-encoder re-ranking | 5 | Re-score fused candidate set, apply similarity floor |
| Deduplication | 5 | Remove overlapping chunks before context assembly |
| Validation | 2, 3 | Re-run both evaluation tracks, compare to hybrid snapshot |

**Rationale:** Re-ranking and deduplication deliver precision and context efficiency gains, but depend on having a broader candidate set (from hybrid retrieval) to re-rank. Must come after P1.2.

### Boundaries

- **In scope:** Logging, evaluation infrastructure, BM25 retrieval, RRF fusion, cross-encoder re-ranking, deduplication
- **Out of scope:** Chat history (P1.1), query rewriting (P1.4), context budgeting (P1.5), UI changes, data pipeline changes

---

## Appendix

### FF v1.1 retrieval pipeline flow

How the 5 epics connect in the request path, showing changes from MVP:

```
FE (Chat UI)
    |
    |  POST /messages
    |
    v
[Session Management & API Layer]  ◄── (MVP — unchanged)
    |
    |  Stores user message in `messages` table
    |
    v
┌─────────────────────────────────────────────────────┐
│  RETRIEVAL                                          │
│                                                     │
│  Query text                                         │
│    │                                                │
│    ├──► Dense retrieval (MVP)                       │
│    │    Embedding API → cosine search → top-k       │
│    │                                                │
│    ├──► BM25 retrieval (Epic 4 — NEW)               │
│    │    Keyword search against prepared keywords     │
│    │                                                │
│    v                                                │
│  Reciprocal Rank Fusion (Epic 4 — NEW)              │
│    Combine dense + BM25 → single ranked list        │
│    (top-20 candidates)                              │
│    │                                                │
│    v                                                │
│  Cross-encoder re-ranking (Epic 5 — NEW)            │
│    Re-score each (query, chunk) pair                │
│    │                                                │
│    v                                                │
│  Deduplication (Epic 5 — NEW)                       │
│    Remove semantically overlapping chunks           │
│    │                                                │
│    v                                                │
│  Similarity floor filter (Epic 5 — NEW)             │
│    Keep only chunks above empirical threshold       │
│    (dynamic chunk count per query)                  │
│                                                     │
└──────────────────────┬──────────────────────────────┘
                       │
                       │  ┌─────────────────────────┐
                       │  │ Epic 1: retrieval_orchestration_logs   │
                       ├─►│ (NEW table)              │
                       │  │ query, k, chunk IDs,     │
                       │  │ similarity scores        │
                       │  └─────────────────────────┘
                       │
                       v
┌─────────────────────────────────────────────────────┐
│  CONTEXT ASSEMBLY & GENERATION  ◄── (MVP — updated) │
│                                                     │
│  Assemble chunks with delimiters + source metadata  │
│  Build prompt (system + context + query)            │
│  Send to LLM via BAML                               │
│    │                                                │
│    v                                                │
│  LLM response includes:                             │
│    - Answer text                                    │
│    - Sources cited                                  │
│    - Relevance confidence: high / mid / low (NEW)   │
│      (BAML schema extended — Epic 1)                │
│                                                     │
└──────────────────────┬──────────────────────────────┘
                       │
                       │  ┌─────────────────────────┐
                       │  │ Epic 1: retrieval_orchestration_logs   │
                       ├─►│ (append to same row)     │
                       │  │ LLM confidence,          │
                       │  │ sources cited             │
                       │  └─────────────────────────┘
                       │
                       v
[Session Management]  ◄── (MVP — unchanged)
    |
    |  Stores assistant response in `messages` table
    |  Returns response to FE
    |
    v
FE (Chat UI)


OFFLINE EVALUATION (Epics 2 & 3)
─────────────────────────────────

retrieval_orchestration_logs table
    │
    ├──► Epic 2: LLM-Derived Baseline (label-free)
    │    - Low confidence count / total queries
    │    - Similarity score → confidence mapping
    │    - Empirical breakpoint calibration
    │
    ├──► Epic 3: Labeled Test Set + IR Metrics
    │    - Synthetic test set from SGDSG service
    │    - Recall@k, Precision@k, MRR
    │    - Compare snapshots across pipeline changes
    │
    v
Baseline snapshots (versioned)
```

**Request flow for `POST /messages` (FF v1.1):**
1. FE calls `POST /messages` with `session_id` + user query
2. Session layer validates `session_id`, stores user message in `messages`
3. **Dense retrieval:** Embedding API converts query → vector, cosine search → top-k
4. **BM25 retrieval (NEW):** Keyword search against prepared keywords → top-k
5. **RRF fusion (NEW):** Combine dense + BM25 → ranked list of top-20 candidates
6. **Cross-encoder re-ranking (NEW):** Re-score top-20 → re-ordered list
7. **Deduplication (NEW):** Remove overlapping chunks
8. **Similarity floor (NEW):** Filter to chunks above threshold (dynamic count)
9. Log retrieval data to `retrieval_orchestration_logs` (query, chunks, scores)
10. Context assembly builds prompt → sends to LLM via BAML → receives response with `relevance_confidence`
11. Log LLM confidence + sources cited to `retrieval_orchestration_logs`
12. Session layer stores assistant response in `messages`, returns to FE

### RRF fusion vs. cross-encoder re-ranking

These are different stages that stack sequentially, not alternatives:

```
              RETRIEVAL                          REFINEMENT
         ┌─────────────────┐              ┌─────────────────────┐
         │                 │              │                     │
Query ──►│  Dense (MVP)    │──┐           │  Cross-encoder      │
         │  cosine search  │  │           │  re-ranking         │
         │  → top-k        │  │           │  (Epic 5)           │
         │                 │  ├──► RRF ──►│                     │──► Dedup ──► Floor ──► Context
         │  BM25 (Epic 4)  │  │  (Epic 4) │  Re-scores each     │    (Epic 5)  (Epic 5)  Assembly
         │  keyword search │  │  Fuses    │  (query, chunk)     │
         │  → top-k        │──┘  into     │  pair individually  │
         │                 │     top-20   │  → re-ordered list  │
         └─────────────────┘              └─────────────────────┘
                 │                                  │
                 │                                  │
          Decides WHAT's in              Decides WHAT's BEST
          the candidate set              in that candidate set
```

- **RRF (Epic 4)** is a **fusion** technique — it interleaves two result sets (dense + BM25) into one ranked list using reciprocal rank scoring. It broadens the candidate pool.
- **Cross-encoder (Epic 5)** is a **re-ranker** — it reads each (query, chunk) pair and produces a new relevance score. It sharpens precision on the fused candidates.

### Logging enums: stage and method

Two-tiered enum design for `retrieval_orchestration_logs`. Stage is stable (pipeline architecture), method captures the implementation (swappable).

**`PipelineStage`** — where in the pipeline the log was captured:

| Value | Description | Epic |
|---|---|---|
| `dense_retrieval` | Vector similarity search | MVP |
| `sparse_retrieval` | Keyword-based search (BM25 today, SPLADE future) | Epic 4 |
| `fusion` | Combine multiple retrieval paths (RRF today, learned fusion future) | Epic 4 |
| `reranking` | Re-score candidates for precision (cross-encoder today) | Epic 5 |
| `deduplication` | Remove overlapping chunks | Epic 5 |
| `similarity_floor` | Filter chunks below relevance threshold | Epic 5 |
| `generation` | LLM response with relevance confidence | Epic 1 |

**`RetrievalMethod`** — which implementation was used at that stage:

| Value | Stage(s) | Status |
|---|---|---|
| `cosine` | `dense_retrieval` | Current (MVP) |
| `bm25` | `sparse_retrieval` | This PRD (Epic 4) |
| `rrf` | `fusion` | This PRD (Epic 4) |
| `cross_encoder` | `reranking` | This PRD (Epic 5) |
| `splade` | `sparse_retrieval` | Future (P3) |
| `colbert` | `reranking` | Future (P3) |
| `learned_fusion` | `fusion` | Future (P3) |

**Example log entry (JSON):**

```json
{
  "message_id": "msg_abc123",
  "stage": "sparse_retrieval",
  "method": "bm25",
  "chunk_ids": ["chunk_042", "chunk_043", "chunk_108"],
  "scores": [0.82, 0.71, 0.65],
  "k": 5,
  "timestamp": "2026-04-17T14:30:00Z"
}
```

Stage tells you *where* in the pipeline, method tells you *how*. Stage enum is stable across implementation changes; method enum grows as new implementations are added. Since the detail is stored as JSON, no schema migration needed — just extend the enum.

### Retrieval model spectrum

Reference for retrieval approaches from simplest to most accurate, including P3 candidates.

| Model | Type | How it works | Tradeoff |
|---|---|---|---|
| **BM25** | Sparse, no learning | Fixed keyword matching with term frequency weighting | Fast, interpretable, no semantic understanding |
| **Dense (bi-encoder)** | Single vector per chunk | Encodes query and chunk separately, compares via cosine/L2 | Semantic matching, but misses exact terms |
| **SPLADE** | Learned sparse | Language model generates a sparse vector where each dimension is a vocabulary term; expands to include related terms not in the original text | Best of sparse + semantic, but needs model training/hosting |
| **ColBERT** | Multi-vector, late interaction | Stores per-token embeddings per chunk; at query time computes token-level similarity (each query token vs. each chunk token) and aggregates | Near cross-encoder quality at retrieval speed, but higher storage cost (one embedding per token, not per chunk) |
| **Cross-encoder** | Full pair scoring | Encodes query + chunk together as one input, outputs relevance score directly | Most accurate, but too slow for first-pass retrieval — only viable for re-ranking small candidate sets |

**Where we are in this PRD:** BM25 + dense hybrid with cross-encoder re-ranking (middle of the spectrum). SPLADE and ColBERT are the P3 upgrade path if hybrid + re-ranking proves insufficient.

### Two-stage similarity floor design

The pipeline applies similarity floors at two points, each with a different purpose and strictness:

| Floor | Where | Epic | Purpose | Threshold | Calibration |
|---|---|---|---|---|---|
| **Retrieval-stage floor** | After dense and BM25, before RRF | Epic 4 | Drop clearly irrelevant noise before fusion | **Lenient** — remove obvious garbage only | Set conservatively at first; tighten using Epic 2 score distribution data |
| **Post-reranking floor** | After cross-encoder, before context assembly | Epic 5 | Only pass high-relevance chunks to the LLM | **Strict** — calibrated from confidence data | Ship with static default; tune from live post-reranking data via Epic 2's similarity → confidence analysis |

**Why two floors instead of one:**
- The retrieval floor reduces noise entering RRF, which means a cleaner fused list and fewer pairs for the cross-encoder to re-score (lower latency)
- The post-reranking floor controls the final chunk count reaching the LLM — this is where quality matters most
- Without the retrieval floor, low-scoring chunks from one retriever can dilute the fused list even if the other retriever found good results
- Both floors are calibrated from Epic 2 data, but at different points on the score distribution

**MVP → FF v1.1 progression:**

| | MVP | FF v1.1 |
|---|---|---|
| **Retrieval k** | Fixed at 5 | Configurable per retriever, empirically tuned via evaluation runs |
| **What reaches the LLM** | All 5 chunks, always | Dynamic — only chunks above the post-reranking floor (could be 2, could be 8) |
| **Quality control** | None — whatever top-5 returns | Two-stage filtering: retrieval floor removes noise, post-reranking floor selects the best |