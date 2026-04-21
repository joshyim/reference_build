# Data Model: Technical Insight Navigator

**Purpose**: Collect and surface technical knowledge with source code references from technical repos. Designed for multi-source ingestion — any repo that contains code samples, guides, or technical content.

**Backend**: Neon (Postgres) with `pgvector` extension for embedding storage and similarity search.

## Summary

The data model has three layers:

**Content layer** — the knowledge base that gets indexed and retrieved against.
- **`sources`** — a technical repo that contains knowledge worth indexing (e.g., `ai-that-works/ai-that-works`). One row per repo.
- **`segments`** — Pass 1 output. Topic boundaries identified within an episode — timestamps, topic label, duration. Persisted so the pipeline can resume from Pass 1 if Pass 2 fails.
- **`chunks`** — Pass 2 output. Structured extraction from a segment — summary, key insights, cleaned text. One chunk per segment.
- **`topics`** — hierarchical, expandable topic taxonomy (e.g., "Evals" → "Programmatic Evals"). Topics link to insights. Supports recursive parent-child via `parent_id`.
- **`tags`** — normalized tag labels extracted during Pass 2. Linked to chunks via `chunk_tags` join table.
- **`insights`** — unfurled from `chunks.key_insights[]`. One row per key insight, providing a per-topic retrieval surface with its own embeddings.
- **`code_refs`** — links an insight to a specific code sample. Scoped to a source repo (path, language, what it demonstrates).
- **`facets`** — a typed tag used for sidebar filtering and navigation. Facets are the informal, user-driven layer; topics are the curated, structured layer. Facets can be promoted to topics over time.
- **`embeddings`** — polymorphic table storing vector embeddings for chunks and insights. Decoupled from content rows to support model swaps and re-embedding without schema migration.

**Workflow layer** — tracks pipeline execution and individual processing steps.
- **`pipeline_runs`** — a top-level record of a pipeline execution. Tracks status, timing, and outcome. The `run_id` is returned to the API caller for polling.
- **`run_tasks`** — individual work items within a run (fetch, segmentation, extraction). Enables fine-grained progress reporting and future resumability.

**Session layer** — tracks user interactions with the chat UI.
- **`chat_sessions`** — one row per conversation. Owns the user's current sidebar facet selections.
- **`messages`** — a single chat turn (user or system). System messages carry cited insights and suggested facets.

**Join tables** (for many-to-many relationships):
- `insight_topics` — insights ↔ topics (with `coverage` and `created_at` metadata)
- `chunk_tags` — chunks ↔ tags
- `insight_facets` — insights ↔ facets
- `session_facets` — active sidebar facets per session
- `message_insights` — which insights a response cited
- `message_facets` — which facets a response suggested

**Total: 14 entities + 6 join tables = 20 Postgres tables.**

## Core Entities

### 1. sources
A technical repo that has been indexed for knowledge extraction.

| Field         | Type      | Notes                                                                 |
| ------------- | --------- | --------------------------------------------------------------------- |
| `id`          | uuid      | Default `gen_random_uuid()`                                           |
| `name`        | string    | Display name (e.g., "AI That Works")                                  |
| `repo_url`    | string    | GitHub URL                                                            |
| `description` | string    | What this source covers                                               |
| `metadata`    | jsonb     | Source-specific fields (e.g., episode structure, content conventions) |
| ~~`indexed_at`~~  | ~~timestamp~~ | ~~Last time insights were extracted~~                                     |

### 2. topics
Hierarchical, expandable topic taxonomy. Grows over time based on user interaction.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Default `gen_random_uuid()` |
| `name` | string | Display name |
| `description` | string | 1–2 sentence scope definition |
| `parent_id` | uuid? | FK to topics — null for top-level topics |

### 3. tags
Normalized tag labels extracted during Pass 2. Linked to chunks via `chunk_tags` join table. Tags are the raw extraction output; facets are the curated, user-facing layer.
[comment] can be linked to chunks and insights

| Field   | Type   | Notes                                                |
| ------- | ------ | ---------------------------------------------------- |
| `id`    | uuid   | Default `gen_random_uuid()`                          |
| `label` | string | Tag text (e.g., `agent-architecture`, `reliability`) |
[comment] move this below insights
### 4. segments
Pass 1 output. Topic boundaries identified within an episode by LLM segmentation. Persisted so the pipeline can resume from Pass 1 if Pass 2 fails. Typically 5–13 segments per episode.

| Field             | Type      | Notes                                                                                                                         |
| ----------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `id`              | uuid      | Default `gen_random_uuid()`                                                                                                    |
| `source_id`       | uuid      | FK to sources                                                                                                                 |
| `episode`         | string    | Episode slug (e.g., `2026-01-13-applying-12-factor-principles-to-coding-agent-sdks`)                                          |
| `segment_number`  | int       | Ordering within the episode (1-based)                                                                                         |
| `topic`           | string    | Topic label (e.g., "Consistency vs. Variance in Agentic Workflows")                                                           |
| `timestamp_start` | string?   | Start timestamp (nullable for non-transcript sources)                                                                         |
| `timestamp_end`   | string?   | End timestamp (nullable for non-transcript sources)                                                                           |
| `created_at`      | timestamp | When Pass 1 produced this row                                                                                                 |

### 5. chunks
Pass 2 output. Structured extraction from a segment — summary, key insights, cleaned text. One chunk per segment. Episode/timestamp context lives on the parent segment (join via `segment_id`).

| Field                | Type      | Notes                                                                              |
| -------------------- | --------- | ---------------------------------------------------------------------------------- |
| `id`                 | uuid      | Default `gen_random_uuid()`                                                        |
| `segment_id`         | uuid      | FK to segments                                                                     |
| `summary`            | string    | 1–3 sentence summary of the segment                                                |
| `raw_text`           | string?   | Cleaned transcript text (optional — kept for transcript-derived chunks)            |
| `search_vector`      | tsvector  | Generated from `summary`, `key_insights`. GIN-indexed for full-text keyword search |
| `created_at`         | timestamp | When Pass 2 produced this row                                                      |
| `timestamp_start_at` | timestamp | chunk start timestamp                                                              |
| `timestamp_end_at`   | timestamp | chunk end timestamp                                                                |

### 6. insights
Unfurled from  chunks: returned as children objects when a chunk was processed. Save one row per key insight. Provides a per-topic retrieval surface with its own embeddings and full-text search. Populated as part of the prep cycle after Pass 2.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Default `gen_random_uuid()` |
| `chunk_id` | uuid | FK to chunks |
| `insight` | string | The key insight text, verbatim from `chunks.key_insights[]` |
| `search_vector` | tsvector | Generated from `insight`. GIN-indexed for full-text keyword search |
[comment] 6.5 insights are children objects of chunks. so we need to create an intersection table between chunks and insights
### 7. insight_topics (join table)
Links insights to topics. Carries metadata because relationships are established and reinforced over time, not just at ingestion.

| Field          | Type       | Notes                                     |
| -------------- | ---------- | ----------------------------------------- |
| `insight_id`   | uuid       | FK to insights                            |
| `topic_id`     | uuid       | FK to topics                              |
| `created_at`   | timestamp  | When this link was established            |

### 8. code_refs
Links an insight to a specific code sample in a source repo.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Default `gen_random_uuid()` |
| `insight_id` | uuid | FK to insights |
| `path` | string | Repo-relative file path |
| `language` | string | `python`, `typescript`, `go`, `baml`, etc. |
| `description` | string | What the sample demonstrates |

### 9. embeddings
Polymorphic table storing vector embeddings for chunks and insights. Decoupled from content rows so embedding models can be swapped or re-run without schema migration.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Default `gen_random_uuid()` |
| `entity_type` | string | `chunk` or `insight` |
| `entity_id` | uuid | FK to the target row (chunks.id or insights.id) |
| `model` | string | Embedding model used (e.g., `text-embedding-3-small`) |
| `dimension` | int | Vector dimension (e.g., 1536) |
| `vector` | vector | pgvector column — dimension varies by model |
| `created_at` | timestamp | When this embedding was generated |

### 10. facets
The clickable tags that flow between chat responses and the sidebar. Facets are the informal sensing layer — they capture what users care about before it's formalized into the topic hierarchy.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Default `gen_random_uuid()` |
| `label` | string | Display text for the chip/pill |
| `type` | string | e.g., `topic`, `pattern`, `technology`, `content-type` |
| `linked_topic_id` | uuid? | FK to topics — set when a facet is promoted to or linked to a topic |

### 11. pipeline_runs
Top-level record of a pipeline execution. Created when a user triggers a run via API. Tracks overall status, timing, and outcome.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Default `gen_random_uuid()` — returned as `run_id` to the caller |
| `source_id` | uuid? | FK to sources (nullable until source is resolved) |
| `transcript_url` | string | The input URL to fetch the transcript from |
| `episode_id` | string? | Derived episode identifier |
| `status` | string | `pending` → `processing` → `success` / `fail` |
| `segments_count` | int? | Populated on completion |
| `chunks_count` | int? | Populated on completion |
| `error_message` | string? | Populated on failure |
| `duration_seconds` | float? | Total processing time |
| `created_at` | timestamp | Default `now()` |
| `completed_at` | timestamp? | Set on completion |

### 12. run_tasks
Individual work items within a pipeline run. Each run has a fixed set of ordered tasks. Enables fine-grained progress reporting and future resumability.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Default `gen_random_uuid()` |
| `run_id` | uuid | FK to pipeline_runs |
| `task_name` | string | `fetch`, `segmentation`, `extraction` |
| `task_order` | int | Execution order (1, 2, 3) |
| `status` | string | `pending` → `processing` → `success` / `fail` |
| `error_message` | string? | |
| `started_at` | timestamp? | |
| `completed_at` | timestamp? | |
| `metadata` | jsonb? | Task-specific output (e.g., `{"segments_found": 8}`) |

### 13. chat_sessions
Tracks one user conversation + their sidebar state.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Default `gen_random_uuid()` |
| `created_at` | timestamp | |
| `active_facets` | Facet[] | Current sidebar selections |
| `messages` | Message[] | Conversation history |

### 14. messages
A single turn in a chat session.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid | Default `gen_random_uuid()` |
| `session_id` | uuid | FK to chat_sessions |
| `role` | string | `user` or `system` |
| `content` | string | The response text |
| `cited_insights` | Insight[] | Which insights were surfaced |
| `suggested_facets` | Facet[] | The 3–5 chips shown below the response |
| `timestamp` | timestamp | |

## Key Relationships

```
pipeline_runs ──< one-to-many  >── run_tasks       (work items within a run)
pipeline_runs ──< one-to-many  >── segments        (rows produced by this run)
pipeline_runs ──< one-to-many  >── chunks          (rows produced by this run)
sources    ──< one-to-many  >── segments        (Pass 1 output)
segments   ──< one-to-one   >── chunks          (Pass 2 output)
chunks     ──< one-to-many  >── insights        (unfurled key_insights)
chunks     ──< many-to-many >── tags            (via chunk_tags)
insights   ──< one-to-many  >── code_refs
embeddings ──< polymorphic  >── chunks/insights (entity_type + entity_id)
insights   ──< many-to-many >── topics          (via insight_topics, with coverage metadata)
insights   ──< many-to-many >── facets          (via insight_facets)
topics     ──< self-ref     >── topics          (parent_id for hierarchy)
facets     ──< optional-FK  >── topics          (linked_topic_id, set on promotion)
chat_sessions ──< one-to-many  >── messages
chat_sessions ──< many-to-many >── facets       (active sidebar)
messages   ──< many-to-many >── insights        (citations)
messages   ──< many-to-many >── facets          (suggestions)
```

## Retrieval Flow (Hybrid RAG with Re-ranking)

1. User sends a message → stored as `Message(role=user)`
2. **Facet pre-filter** — narrow candidate pool by `chat_sessions.active_facets` (exact match via join tables)
3. **Parallel retrieval** on filtered candidates:
   - **Embedding search** — cosine similarity via `embeddings` table against chunks and/or insights
   - **Keyword search** — full-text `tsquery` against `search_vector` on chunks and/or insights
4. **Merge** candidate sets from both paths (union, deduplicate)
5. **Re-rank** merged candidates (cross-encoder model or similar) to produce final Top-N
6. Response `Message` created with `cited_insights` and `suggested_facets`
7. User clicks facets → added to `chat_sessions.active_facets`

## Postgres Notes

- `CREATE EXTENSION vector;` on Neon for pgvector
- `embeddings.vector` indexed with `ivfflat` or `hnsw` for approximate nearest neighbor search
- `search_vector` columns indexed with GIN for full-text keyword search
- Embedding model choice (e.g., OpenAI `text-embedding-3-small` at 1536d) determines vector dimension — stored per-row in `embeddings.dimension`
- Re-ranking is application-layer (not in Postgres) — operates on the merged candidate set

## Open Design Decisions

- **Insight granularity** — [resolved] Pass 2 produces chunks with `key_insights[]`. These are unfurled into the `insights` table — one row per key insight — with per-topic embeddings for fine-grained retrieval.
- **Learning paths** — ordered sequences per topic would need a `LearningPath` entity with ordered steps
- **Multi-source dedup** — what happens when two sources cover the same topic? Cross-reference at chunk or insight level?
- **Tags → facets promotion** — `tags` are raw extraction output linked to chunks. How/when do they get promoted into the curated `facets` table? Manual curation, or auto-seed on ingest?
- **Re-ranker choice** — cross-encoder (e.g., Cohere Rerank, BGE reranker) vs. custom scoring function. Tradeoff between quality and latency/cost.
