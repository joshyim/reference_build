## 📋 Implementation Plan

> Exported from Antigravity conversation `f5520b3c-e915-4422-96de-4b66846ce9fa`

# Epic 4: BM25 Retrieval + Hybrid Fusion (ATW-47)

> **Goal:** Add keyword-based retrieval (BM25 via Postgres `tsvector`/`tsquery`), combine it with the existing dense retrieval via Reciprocal Rank Fusion (RRF), and replace the dense-only result set in the pipeline — all with configurable per-retriever top-k and a lenient retrieval-stage similarity floor.

## User Review Required

> \[!NOTE\]
> **Configurable k defaults** — Starting at `k=10` for both dense and BM25 retrievers. Experiments will run at k=10, 20, 30 to empirically tune the optimal value per retriever.

> \[!NOTE\]
> **Similarity floor defaults** — Starting lenient (`0.2` for dense cosine, `0.5` for BM25 `ts_rank`). Will tighten based on empirical results from evaluation runs.

> \[!NOTE\]
> **RRF constant** — Using the standard `k=60` from the original RRF paper. Widely adopted default.

> \[!NOTE\]
> **Architecture** — `BM25RetrievalService`, `HybridRetrievalService`, and the existing `VectorRetrievalService` all live under `app/services/`. `ChatService` swaps its dependency from `VectorRetrievalService` → `HybridRetrievalService`.

---

## Proposed Changes

### Configuration Layer

#### \[MODIFY\] \[[config.py](<http://config.py>)\](file:///Users/joshyim/projects/atw_nav/backend/app/core/config.py)

Add retrieval configuration settings to `Settings`:

```python
# Retrieval Configuration
dense_retrieval_k: int = 10
bm25_retrieval_k: int = 10
dense_similarity_floor: float = 0.2
bm25_similarity_floor: float = 0.5
rrf_k_constant: int = 60
rrf_dense_weight: float = 1.0
rrf_bm25_weight: float = 1.0
```

All values configurable via environment variables (e.g., `DENSE_RETRIEVAL_K=20`).

---

### Schema Layer

#### \[MODIFY\] \[[retrieval.py](<http://retrieval.py>)\](file:///Users/joshyim/projects/atw_nav/backend/app/schemas/retrieval.py)

* Add `retrieval_method` field to `RetrievalResult` so downstream can distinguish the origin of each chunk:

```python
class RetrievalResult(BaseModel):
    chunk_id: UUID
    content: str
    similarity_score: float
    source_metadata: SourceMetadata
    retrieval_method: Optional[str] = None  # "cosine", "bm25", "rrf"
```

* Add new `FusedRetrievalResult` model with an `rrf_score` field used after fusion:

```python
class FusedRetrievalResult(BaseModel):
    chunk_id: UUID
    content: str
    rrf_score: float
    dense_score: Optional[float] = None
    bm25_score: Optional[float] = None
    source_metadata: SourceMetadata
```

#### \[MODIFY\] \[[logging.py](<http://logging.py>)\](file:///Users/joshyim/projects/atw_nav/backend/app/schemas/logging.py)

* Add `floor_threshold` and `candidates_before_floor` optional fields to `RetrievalLogAttrs` so floor filtering is observable in logs:

```python
class RetrievalLogAttrs(BaseModel):
    ...  # existing fields
    floor_threshold: Optional[float] = None
    candidates_before_floor: Optional[int] = None
    rrf_k_constant: Optional[int] = None
    dense_weight: Optional[float] = None
    bm25_weight: Optional[float] = None
```

---

### Service Layer

#### \[MODIFY\] \[vector_retrieval_service.py\](file:///Users/joshyim/projects/atw_nav/backend/app/services/vector_retrieval_service.py)

* Accept `k` from caller (already does via parameter).
* Populate `retrieval_method="cosine"` on each returned `RetrievalResult`.
* No other changes — this service remains the dense-only retriever.

#### \[NEW\] \[bm25_retrieval_service.py\](file:///Users/joshyim/projects/atw_nav/backend/app/services/bm25_retrieval_service.py)

New service for keyword-based BM25 retrieval against the existing `search_vector` tsvector column on the `chunks` table:

```
class BM25RetrievalService:
    async def retrieve_chunks(query: str, conn: asyncpg.Connection, k: int) -> List[RetrievalResult]
```

* Uses `plainto_tsquery('english', $1)` to parse the user query.
* Runs `ts_rank(c.search_vector, query)` for scoring.
* JOINs to `segments` → `sources` for metadata (same pattern as vector service).
* Returns empty list gracefully when no keyword matches exist.
* Populates `retrieval_method="bm25"` on results.

SQL pattern:

```sql
SELECT c.id AS chunk_id, c.raw_text AS content,
       ts_rank(c.search_vector, plainto_tsquery('english', $1)) AS bm25_score,
       s.name AS source_name, s.url, seg.timestamp_start
FROM chunks c
JOIN segments seg ON c.segment_id = seg.id
JOIN sources s ON seg.source_id = s.id
WHERE c.search_vector @@ plainto_tsquery('english', $1)
ORDER BY bm25_score DESC
LIMIT $2;
```

> \[!NOTE\]
> `ts_rank` scores are unbounded — while they are always ≥ 0, there is no upper bound of 1.0. Scores depend on document length, term frequency, and weighting configuration. Short documents with few matches tend to produce small values (often < 1.0), but longer documents with dense keyword matches can exceed 1.0. Since RRF is rank-based and ignores absolute scores, this unboundedness doesn't affect fusion. The floor threshold (`0.5` default) is applied to raw `ts_rank` values before fusion to drop obvious garbage.

#### \[NEW\] \[hybrid_retrieval_service.py\](file:///Users/joshyim/projects/atw_nav/backend/app/services/hybrid_retrieval_service.py)

Orchestrates both retrievers + floor + RRF fusion:

```
class HybridRetrievalService:
    def __init__(
        self,
        vector_service: VectorRetrievalService,
        bm25_service: BM25RetrievalService,
        settings: Settings,
    )

    async def retrieve_and_fuse(
        query: str, conn: asyncpg.Connection
    ) -> (List[FusedRetrievalResult], dict[str, Any])
```

**Responsibilities:**

1. Run dense retrieval (`k=settings.dense_retrieval_k`)
2. Run BM25 retrieval (`k=settings.bm25_retrieval_k`)
3. Apply retrieval-stage similarity floor to each set independently (drop chunks below dense floor / BM25 floor)
4. Apply Reciprocal Rank Fusion: `score(d) = Σ (weight / (k_constant + rank))` across both lists
5. Deduplicate by `chunk_id` (same chunk from both retrievers gets summed RRF scores)
6. Return the fused list sorted by descending RRF score
7. Return a metadata dict capturing logging info (e.g., `candidates_before_floor`, `candidates_after_floor`, etc.)

**RRF formula:**

```
rrf_score(chunk) = dense_weight / (k + rank_dense) + bm25_weight / (k + rank_bm25)
```

Where `rank_dense` and `rank_bm25` are the 1-indexed positions of the chunk in each retriever's result (or `∞` if absent from that retriever, contributing 0 to the sum).

---

### Chat Service Integration

#### \[MODIFY\] \[chat_service.py\](file:///Users/joshyim/projects/atw_nav/backend/app/services/chat_service.py)

**Key changes:**

1. Replace `VectorRetrievalService` dependency → `HybridRetrievalService`
2. Call `hybrid_service.retrieve_and_fuse(...)` instead of `retrieval_service.retrieve_chunks(...)`
3. Map `FusedRetrievalResult` → context assembly (same format: `--- Chunk [N] ---`)
4. Log **three** pipeline stages instead of one:
   * `dense_retrieval` / `cosine` — Dense results + floor info
   * `sparse_retrieval` / `bm25` — BM25 results + floor info
   * `fusion` / `rrf` — Fused results + RRF params
5. The final chunk count reaching context assembly is now the RRF fused list size. Epic 5's post-reranking floor controls what reaches the LLM — until Epic 5 is implemented, all fused chunks reach context assembly.

#### \[MODIFY\] \[[session.py](<http://session.py>)\](file:///Users/joshyim/projects/atw_nav/backend/app/api/session.py)

* Update `get_chat_service` DI factory to construct `HybridRetrievalService` wrapping `VectorRetrievalService` and `BM25RetrievalService`, passing `settings`.

---

### Test Layer

#### \[NEW\] \[test_bm25_retrieval.py\](file:///Users/joshyim/projects/atw_nav/backend/test/test_bm25_retrieval.py)

Unit tests for `BM25RetrievalService`:

* Correct SQL query construction with `plainto_tsquery`
* Empty query returns empty list
* Empty result set from DB handled gracefully
* `retrieval_method="bm25"` populated on all results
* Score and metadata mapping from DB records

#### \[NEW\] \[test_hybrid_retrieval.py\](file:///Users/joshyim/projects/atw_nav/backend/test/test_hybrid_retrieval.py)

Unit tests for `HybridRetrievalService`:

* RRF score calculation with known inputs (deterministic, verifiable by hand)
* Floor filtering removes chunks below threshold for both retrievers
* Deduplication merges same chunk from both retrievers (summed RRF scores)
* One retriever returns empty → fusion degrades gracefully to single-retriever results
* Both retrievers return empty → returns empty list
* Metadata dict captures `candidates_before_floor`, `candidates_after_floor`

Tests will use the same patterns as existing tests in `test/` — pytest with `unittest.mock.AsyncMock` for mocking `conn.fetch` and service dependencies.

---

### Evaluation & Validation

#### \[MODIFY\] \[eval_ir_metrics.py\](file:///Users/joshyim/projects/atw_nav/backend/evals/scripts/eval_ir_metrics.py)

* Add `--mode` flag: `dense` (default for backward compat) or `hybrid`.
* In `hybrid` mode, use `HybridRetrievalService.retrieve_and_fuse()` and evaluate the fused result set.
* Snapshot output includes a `mode` field so dense-only vs. hybrid snapshots are clearly labeled.

#### \[MODIFY\] \[eval_baseline.py\](file:///Users/joshyim/projects/atw_nav/backend/evals/scripts/eval_baseline.py)

* The existing code already handles `sparse_retrieval` and `fusion` log stages (lines 98-116) — no structural changes needed.
* Metrics will auto-populate once hybrid retrieval logs are generated.

---

## Implementation Order

| Step | User Story | Files | Depends On |
| -- | -- | -- | -- |
| 1 | Configuration | `config.py` | — |
| 2 | Schemas | `retrieval.py`, `logging.py` | Step 1 |
| 3 | BM25 Retrieval | `bm25_retrieval_service.py` (NEW) | Steps 1–2 |
| 4 | Modify Dense Service | `vector_retrieval_service.py` | Step 2 |
| 5 | Hybrid + RRF | `hybrid_retrieval_service.py` (NEW) | Steps 3–4 |
| 6 | Unit Tests | `test_bm25_retrieval.py`, `test_hybrid_retrieval.py` (NEW) | Steps 3–5 |
| 7 | Chat Integration | `chat_service.py`, `session.py` | Step 5 |
| 8 | Eval Updates | `eval_ir_metrics.py` | Step 5 |
| 9 | Validation Runs | — | Steps 7–8 |

---

---

## Verification Plan

### Automated Tests

Run existing + new unit tests:

```bash
uv run pytest test/test_bm25_retrieval.py test/test_hybrid_retrieval.py -v
```

### Integration / Evaluation

1. **Baseline snapshot** — Run `eval_ir_metrics.py` in `dense` mode at k=10 → save snapshot
2. **Hybrid experiments** — Run `eval_ir_metrics.py` in `hybrid` mode at k=10, 20, 30 → save snapshots
3. **Regression comparison** — Compare hybrid vs. dense using `--compare` flag — confirm no regression on Recall@k, Precision@k, MRR
4. **Logging validation** — Run `eval_baseline.py` to confirm `sparse_retrieval` and `fusion` log stages are populated in `retrieval_orchestration_logs`
5. **Manual smoke test** — Chat queries with exact-match terminology (product names, acronyms) to verify BM25 improves results over dense-only