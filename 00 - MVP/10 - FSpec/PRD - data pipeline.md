---
status: in-progress
---
# Objective
Take in episode data from [AI That Works repo](https://github.com/ai-that-works/ai-that-works) then retrieve transcript from each episode. Right now only a few of them has transcripts. So, only focus on digesting them. Converting video to transcripts will take place later (out of scope for the current efforts). Then apply [two pass methods](Tasks/atw_nav/20 - Ideation/transcript_chunking_strategy) to populate segments, chunks, and insights tables, as well as ancillary tables such as topics, tags, and insight_topics (join table).
Take the sources table as the starting point of the process. In this case will only contain single repo - AI That Works repo. From the repo, the process will need to detect new episodes, then schedule processing job (P2)

---

## 1. Executive Summary

Build a data pipeline that ingests episode content from the AI That Works GitHub repo, applies a two-pass LLM extraction process (segmentation → structured extraction), and populates a Neon Postgres database with segments, chunks, insights, topics, tags, and their relationships. The pipeline enables a downstream chat-based insight navigator by producing the structured knowledge base it retrieves against.

The initial scope is limited to the ~7 episodes that already have transcripts. Video-to-transcript conversion and non-transcript episode ingestion are deferred.

---

## 2. Product Overview

### Purpose
The data pipeline is the ingestion backbone of the Technical Insight Navigator. It transforms raw podcast transcripts into structured, searchable knowledge — segments with topic boundaries, chunks with summaries and key insights, and normalized topics and tags — stored in Postgres with full-text search indexes and embedding-ready rows.

### Key Features
- **Source registration** — register a GitHub repo as a knowledge source (OOS)
- **Episode detection** — scan a registered source for episodes with available transcripts
- **Two-pass LLM extraction** — Pass 1 segments transcripts into topical blocks; Pass 2 extracts structured chunks with summaries, key insights, speakers, and tags
- **Insight unfurling** — expand `chunks.key_insights[]` into individual `insights` rows for fine-grained retrieval
- **Topic and tag normalization** — deduplicate and link extracted tags to chunks, and map insights to topics
- **Idempotent re-processing** — re-running the pipeline on an already-processed episode does not create duplicates

### Business Alignment
This pipeline is the prerequisite for the hybrid RAG retrieval system described in the [data model](Tasks/atw_nav/40 - Design/data-model.md). Without structured, indexed content in the database, the chat UI and search layer have nothing to query.

---

## 3. Target Personas

### Pipeline Operator
- **Role**: Runs and monitors the pipeline; diagnoses failures
- **Pain points**: Needs clear logging and resumability — if Pass 2 fails on segment 5 of 8, don't redo segments 1–4
- **Motivation**: Get high-quality structured data into the DB with minimal manual intervention

---

## 4. Functional Requirements (Epics)

### Epic 1: Source Registration and Episode Discovery — On Hold

> As a **pipeline operator**, I want to register a GitHub repo as a source and discover which episodes have transcripts, so that I know what content is available for processing.

**Status**: On hold. The only source is AI That Works for now. Different repos will have different structures, so generalizing source registration is deferred until a second source is needed.

**In Scope** (when resumed):
- Insert or update a row in the `sources` table for the AI That Works repo
- Scan the repo's episode folders (pattern: `YYYY-MM-DD-episode-name/`) for the presence of `transcript.md`
- Produce a manifest of episodes ready for processing (episode slug, transcript path, whether already processed)
- Skip episodes that have already been fully processed (segments + chunks exist)

**Out of Scope**:
- Cloning/pulling the repo automatically (assume a local clone or API access is pre-configured)
- Processing episodes without transcripts (README-only, meta-only)
- Webhook-based detection of new episodes (P2 — scheduling)

---

### Epic 2: Pipeline API

> As a **pipeline operator**, I want an API that accepts a transcript file path and triggers the processing flow, so that the frontend can initiate pipeline runs.

**In Scope**:
- **API entrypoint**: expose the pipeline as an API (e.g. FastAPI) callable from the frontend, accepting a transcript file path as a required parameter
- Request validation: verify the transcript path exists and is a supported format
- Invoke the pipeline state machine (Epic 3) to execute the processing steps
- Return processing status and results to the caller
- Logging: log each request's start/end, episode being processed, error details

**Out of Scope**:
- Internal step sequencing and resumability (see Epic 3)
- Scheduled/cron-based execution (P2)
- Parallel processing of multiple episodes (process sequentially for now)
- Web UI or dashboard for monitoring

---

### Epic 3: Pipeline State Machine and Resumability

> As a **pipeline operator**, I want the pipeline to sequence processing steps and resume from the last successful step on failure, so that I don't re-process completed work.

**In Scope**:
- Orchestrate the processing steps: Pass 1 (segmentation) → Pass 2 (chunk extraction) → insight unfurling & tag/topic linking → embedding generation
- Track per-episode step completion state (e.g. which steps have been successfully run)
- Per-episode resumability: if an earlier step succeeds but a later step fails, the next run picks up from the failed step using persisted intermediate results
- State persistence: persist information and state of processing task and workers so that it can be resumed or re-processed
- Step-level logging: log each step's start/end, segment count, chunk count, error details
- Dry-run mode: optional flag that logs what would be processed without writing to the DB

**Out of Scope**:
- Retry with backoff for LLM API failures (fail fast, re-run manually)
- Parallel step execution (steps run sequentially)
- Step dependency graph beyond the linear chain

---

### Epic 4: Pass 1 — Transcript Segmentation

> As a **pipeline operator**, I want to segment a transcript into topical blocks with timestamps and labels, so that each block can be independently extracted in Pass 2.

**In Scope**:
- **Required input**: the file path to the transcript is provided by the caller (via the Epic 2 API)
- For each episode transcript, run LLM segmentation to identify 5–13 topic boundaries
- Each segment includes: topic label, start/end timestamps, 1-sentence summary, segment ordering number
- Persist segments to the `segments` table with `source_id`, `episode`, `episode_title`, `segment_number`
- Filter out intro banter, logistics, and sub-30-second tangents per the [chunking strategy](Tasks/atw_nav/20 - Ideation/transcript_chunking_strategy)
- Idempotent: re-running Pass 1 on an episode replaces its segments (upsert by episode + segment_number)

**Out of Scope**:
- Sliding window / context overflow handling (transcripts are ~82K chars, within current LLM context limits)
- Mechanical fallback chunking (5-minute windows) — only the LLM approach is in scope for now

---

### Epic 5: Pass 2 — Structured Chunk Extraction

> As a **pipeline operator**, I want to extract structured chunks from each segment, so that the knowledge base has summaries, key insights, speakers, and cleaned text for retrieval.

**In Scope**:
- For each segment produced by Pass 1, run LLM extraction to produce a chunk
- Each chunk includes: `speakers[]`, `summary`, `key_insights[]`, `raw_text` (cleaned transcript), tags
- Apply cleaning rules: drop backchannels, merge split turns (<5s same speaker), strip logistics, preserve verbatim key quotes, maintain speaker attribution
- Respect chunk sizing: target 1,000–3,000 tokens; merge segments under 500 tokens with adjacent; split segments over 5,000 tokens at sub-topic breaks
- Include last 2 turns of prior segment as overlap context
- Persist chunks to the `chunks` table linked to their parent segment
- Generate and store `search_vector` (tsvector) from summary + key_insights on insert
- Idempotent: re-running Pass 2 replaces chunks for the given segments

**Out of Scope**:
- Embedding generation (handled separately — see Epic 7)
- Processing non-transcript content (README, whiteboards, journals)

---

### Epic 6: Insight Unfurling and Topic/Tag Linking

> As a **pipeline operator**, I want each key insight stored as its own row and linked to topics and tags, so that downstream search can retrieve at the insight level.

**In Scope**:
- After Pass 2, unfurl `chunks.key_insights[]` into individual rows in the `insights` table (one row per insight, FK to chunk)
- Generate `search_vector` (tsvector) on each insight row
- Extract tags from Pass 2 output; normalize (lowercase, deduplicate) and upsert into `tags` table
- Create `chunk_tags` join rows linking chunks to their tags
- Map insights to topics via `insight_topics` join table with `coverage` metadata (`deep`, `partial`, `tangential`)
- Topic assignment can use LLM-assisted classification against the existing topic taxonomy, or default to a seed set of top-level topics from the [basic research](Tasks/atw_nav/10 - Direction Setting/basic_research.md) thematic index

**Out of Scope**:
- Topic hierarchy expansion (new child topics) — initial run uses a flat seed set
- Facet generation and `insight_facets` linking (deferred to the chat/UI layer)
- `code_refs` population (requires separate code sample indexing)

---

### Epic 7: Embedding Generation

> As a **pipeline operator**, I want vector embeddings generated for chunks and insights, so that semantic search works alongside keyword search.

**In Scope**:
- After chunks and insights are persisted, generate embeddings and store in the `embeddings` table
- Store `entity_type` (`chunk` or `insight`), `entity_id`, `model`, `dimension`, and `vector`
- Support configurable embedding model (default: OpenAI `text-embedding-3-small`, 1536d)
- For chunks: embed the `summary` field
- For insights: embed the `insight` field
- Idempotent: skip rows that already have an embedding for the same model

**Out of Scope**:
- Re-embedding when models change (manual re-run)
- Index creation (ivfflat/hnsw) — handled in DB setup, not the pipeline
- Embedding non-text content (code samples, images)

---

### Cross-Cutting Out of Scope
- **Video-to-transcript conversion** — transcription of episodes without `transcript.md` is a separate future effort
- **Non-transcript episode processing** — ingesting README.md, whiteboards.md, journal.md as chunk sources
- **Chat/session layer** — `chat_sessions`, `messages`, `session_facets`, `message_insights`, `message_facets` tables are populated by the application layer, not this pipeline
- **Facet management** — facets are a user-interaction concept, not a pipeline output
- **Multi-source support** — the pipeline architecture should support multiple sources, but only AI That Works is in scope for initial build
- **New episode auto-detection** — webhook or polling-based detection of newly pushed episodes (P2)

---

## 5. User Stories

### Epic 1: Source Registration and Episode Discovery — On Hold

| Title | Story Detail | Acceptance Criteria | Priority |
|---|---|---|---|
| Register GitHub Source | As a **pipeline operator**, I want to register the AI That Works GitHub repo as a source, so that the pipeline knows where to find episode content. | - A row is inserted/updated in the `sources` table for the repo. <br>- Running registration twice does not create duplicates. | Low (on hold) |
| Discover Episodes with Transcripts | As a **pipeline operator**, I want to scan a registered source for episodes that contain `transcript.md`, so that I know which episodes are ready for processing. | - Scans folders matching `YYYY-MM-DD-episode-name/` pattern. <br>- Returns a manifest with episode slug, transcript path, and processing status. <br>- Episodes already fully processed (segments + chunks exist) are marked as skipped. | Low (on hold) |

**Out of scope**: Auto-cloning/pulling the repo, processing episodes without transcripts, webhook-based new-episode detection.

---

### Epic 2: Pipeline API

| Title                          | Story Detail                                                                                                                                                  | Acceptance Criteria                                                                                                                                                                                                         | Priority |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| Accept Transcript Path via API | As a **pipeline operator**, I want to call an API endpoint with a transcript file path, so that I can trigger pipeline processing from the frontend.          | - API endpoint accepts a transcript file path as a required parameter. <br>- Returns an error if the path does not exist or is an unsupported format. <br>- Successful request invokes the pipeline state machine (Epic 3). | High     |
| Return Processing Status       | As a **pipeline operator**, I want the API to return processing status and results, so that I know whether the run succeeded or failed and what was produced. | - Response includes overall status (success/failure). <br>- Response includes counts (segments, chunks, insights created). <br>- On failure, response includes the step that failed and error details.                      | High     |
| Log API Requests               | As a **pipeline operator**, I want each API request logged with start/end timestamps and episode identifier, so that I can audit pipeline activity.           | - Each request logs: timestamp, transcript path, episode identifier, duration, outcome. <br>- Errors are logged with detail sufficient to diagnose without re-running.                                                      | Medium   |

**Out of scope**: Internal step sequencing and resumability (Epic 3), scheduled/cron execution, parallel episode processing, monitoring dashboard.

---

### Epic 3: Pipeline State Machine and Resumability

| Title                            | Story Detail                                                                                                                                                                                                             | Acceptance Criteria                                                                                                                                                                                                                                                                                                                            | Priority |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| Orchestrate Processing Steps     | As a **pipeline operator**, I want the pipeline to execute steps in order (Pass 1 `segmentation` → Pass 2 `chunk_extraction` → `insight_unfurling` → `embedding_generation`), so that each step's output feeds the next. | - Steps execute in the defined sequence. <br>- Each step receives the output/persisted data from the prior step. <br>- If a step fails, subsequent steps do not execute.                                                                                                                                                                       | High     |
| Track Step Completion            | As a **pipeline operator**, I want per-episode step completion tracked, so that I can see which steps have run successfully.                                                                                             | - Completion state is persisted per episode per step. <br>- State is queryable (e.g., "episode X completed Pass 1 but not Pass 2").                                                                                                                                                                                                            | High     |
| Resume from Last Successful Step | As a **pipeline operator**, I want the pipeline to resume from the last successful step on re-run, so that I don't re-process completed work.                                                                            | - Re-running a partially processed episode skips already-completed steps. <br>- Resumed run uses persisted intermediate results from prior steps. <br>- Resumed run produces the same final output as a clean run.                                                                                                                             | High     |
| Persist Pipeline State           | As a **pipeline operator**, I want the pipeline to persist processing task state and worker information, so that runs can be resumed after crashes or re-processed on demand.                                            | - Task state (current step, step outcomes, timestamps) is persisted durably (not just in-memory). <br>- Intermediate results from each completed step are persisted and available for resumption. <br>- State is queryable to determine which episodes are in-progress, completed, or failed. <br>- Persisted state survives process restarts. | High     |
| Dry-Run Mode                     | As a **pipeline operator**, I want a dry-run flag that logs what would be processed without writing to the DB, so that I can preview a run before committing.                                                            | - Dry-run logs each step and what it would produce (counts, identifiers). <br>- No rows are inserted, updated, or deleted in the database.                                                                                                                                                                                                     | Medium   |
| Step-Level Logging               | As a **pipeline operator**, I want each step to log its start/end time, counts, and errors, so that I can diagnose failures at the step level.                                                                           | - Logs include: step name, start timestamp, end timestamp, segment/chunk/insight counts, error details if failed.                                                                                                                                                                                                                              | Medium   |

**Out of scope**: Retry with backoff for LLM failures, parallel step execution, dependency graph beyond the linear chain.

---

### Epic 4: Pass 1 — Transcript Segmentation

| Title                                | Story Detail                                                                                                                                                               | Acceptance Criteria                                                                                                                                                       | Priority |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| Segment Transcript into Topic Blocks | As a **pipeline operator**, I want a transcript segmented into 5–13 topical blocks via LLM, so that each block can be independently extracted in Pass 2.                   | - Each segment has: topic label, start/end timestamps, 1-sentence summary, segment ordering number. <br>- Segment count is between 5 and 13 per episode.                  | High     |
| Persist Segments                     | As a **pipeline operator**, I want segments persisted to the `segments` table, so that Pass 2 can consume them and they survive pipeline restarts.                         | - Rows written to `segments` with `source_id`, `episode`, `episode_title`, `segment_number`. <br>- Re-running Pass 1 upserts by episode + segment_number (no duplicates). | High     |
| Filter Non-Substantive Content       | As a **pipeline operator**, I want intro banter, logistics, and sub-30-second tangents excluded from segments, so that only substantive content enters the knowledge base. | - Segments do not contain intro banter or logistics per the chunking strategy. <br>- Tangents under 30 seconds are excluded or merged.                                    | Medium   |

**Out of scope**: Sliding window / context overflow handling, mechanical fallback chunking (5-minute windows).

---

### Epic 5: Pass 2 — Structured Chunk Extraction

| Title | Story Detail | Acceptance Criteria | Priority |
|---|---|---|---|
| Extract Structured Chunks from Segments | As a **pipeline operator**, I want each segment processed by LLM to produce a structured chunk, so that the knowledge base has summaries, insights, speakers, and cleaned text. | - Each chunk includes: `speakers[]`, `summary`, `key_insights[]`, `raw_text`, tags. <br>- Cleaning rules applied: backchannels dropped, split turns merged (<5s same speaker), logistics stripped, key quotes preserved verbatim, speaker attribution maintained. | High |
| Enforce Chunk Sizing | As a **pipeline operator**, I want chunks sized between 1,000–3,000 tokens, so that retrieval returns appropriately scoped results. | - Segments under 500 tokens are merged with adjacent segments. <br>- Segments over 5,000 tokens are split at sub-topic breaks. <br>- Last 2 turns of prior segment included as overlap context. | High |
| Persist Chunks with Search Vector | As a **pipeline operator**, I want chunks persisted to the `chunks` table with a `search_vector`, so that keyword search works immediately on insert. | - Rows written to `chunks` linked to parent segment. <br>- `search_vector` (tsvector) generated from summary + key_insights. <br>- Re-running Pass 2 replaces chunks for the given segments (idempotent). | High |

**Out of scope**: Embedding generation (Epic 7), processing non-transcript content.

---

### Epic 6: Insight Unfurling and Topic/Tag Linking

| Title | Story Detail | Acceptance Criteria | Priority |
|---|---|---|---|
| Unfurl Key Insights into Individual Rows | As a **pipeline operator**, I want each entry in `chunks.key_insights[]` stored as its own row in the `insights` table, so that downstream search can retrieve at the insight level. | - One `insights` row per key insight, with FK to parent chunk. <br>- `search_vector` (tsvector) generated on each insight row. | High |
| Normalize and Link Tags | As a **pipeline operator**, I want tags from Pass 2 normalized and linked to chunks, so that tag-based filtering works consistently. | - Tags are lowercased, deduplicated, and upserted into the `tags` table. <br>- `chunk_tags` join rows created linking each chunk to its tags. | High |
| Map Insights to Topics | As a **pipeline operator**, I want insights mapped to topics with coverage metadata, so that topic-based navigation returns relevant insights. | - `insight_topics` join rows created with `coverage` (`deep`, `partial`, `tangential`). <br>- Topic assignment uses LLM classification or seed set from basic research thematic index. | Medium |

**Out of scope**: Topic hierarchy expansion, facet generation and `insight_facets` linking, `code_refs` population.

---

### Epic 7: Embedding Generation

| Title | Story Detail | Acceptance Criteria | Priority |
|---|---|---|---|
| Generate Chunk Embeddings | As a **pipeline operator**, I want vector embeddings generated for each chunk's summary, so that semantic search can find relevant chunks. | - Embedding stored in `embeddings` table with `entity_type='chunk'`, `entity_id`, `model`, `dimension`, `vector`. <br>- Default model: OpenAI `text-embedding-3-small` (1536d). <br>- Idempotent: skips chunks that already have an embedding for the same model. | High |
| Generate Insight Embeddings | As a **pipeline operator**, I want vector embeddings generated for each insight, so that semantic search can find relevant insights. | - Embedding stored in `embeddings` table with `entity_type='insight'`, `entity_id`, `model`, `dimension`, `vector`. <br>- Idempotent: skips insights that already have an embedding for the same model. | High |
| Support Configurable Embedding Model | As a **pipeline operator**, I want the embedding model to be configurable, so that I can switch models without code changes. | - Model name and dimension are configuration parameters. <br>- Changing the model and re-running generates new embeddings without deleting old ones. | Low |

**Out of scope**: Re-embedding on model change (manual re-run), index creation (ivfflat/hnsw), embedding non-text content.

---

## 6. Success Metrics

### Leading Indicators (change quickly)
- **Episodes processed**: 7/7 transcript-available episodes fully processed without manual intervention
- **Segment yield**: 5–13 segments per episode (consistent with [chunking strategy](Tasks/atw_nav/20 - Ideation/transcript_chunking_strategy) expectations)
- **Chunk completeness**: 100% of chunks have non-empty `summary`, `key_insights[]`, and `search_vector`
- **Insight coverage**: every insight links to at least one topic via `insight_topics`

### Lagging Indicators (change over time)
- **Retrieval quality**: hybrid search (keyword + semantic) returns relevant results for test queries against the populated data — target: top-3 results contain a relevant insight for 80%+ of test queries
- **Pipeline stability**: re-running the pipeline on already-processed episodes produces no duplicates and completes in < 2 minutes
- **Incremental processing**: when a new transcript is added to the repo, only that episode gets processed on the next run

---

## 7. Open Questions

| Question                                                             | Owner              | Context                                                                                                                                                |
| -------------------------------------------------------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Which LLM to use for Pass 1 and Pass 2? (Claude, GPT-4, etc.)        | Engineering        | Cost vs. quality tradeoff. The [experiment](Tasks/atw_nav/40 - Experimentation/dual_pass_extraction_test) used Claude — is that the production choice? |
| How should the initial topic seed set be defined?                    | Engineering / Data | Use the 8 themes from [basic research](Tasks/atw_nav/10 - Direction Setting/basic_research.md), or let Pass 2 tags bootstrap topics?                   |
| Should the pipeline run against a local clone or use the GitHub API? | Engineering        | Local clone is simpler but requires manual `git pull`. API avoids that but adds rate limits and complexity.                                            |
| What's the embedding model budget?                                   | Engineering        | `text-embedding-3-small` is cheap (~$0.02/1M tokens) but `text-embedding-3-large` (3072d) may improve retrieval quality.                               |
| How should topic-to-insight mapping work for the initial load?       | Engineering / Data | LLM-assisted classification is higher quality but adds cost. Rule-based tag-to-topic mapping is cheaper but less accurate.                             |
| Where does the pipeline run?                                         | Engineering        | Local script, GitHub Action, or hosted service? Affects secrets management and scheduling.                                                             |

---

## 8. Roadmap

### Now
- **Epics 1–5**: Source registration, episode discovery, two-pass extraction, insight unfurling, tag/topic linking, and pipeline orchestration for the 7 transcript-available episodes
- **Rationale**: This is the minimum to populate the database and unblock downstream search/chat development. Sequential processing and manual runs are acceptable at this scale.

### Next
- **Epic 6**: Embedding generation for chunks and insights
- **Non-transcript episode ingestion**: Process the remaining ~46 episodes using README.md, meta.md, whiteboards.md, and journal.md as source content
- **Rationale**: Embeddings require a model choice decision (open question). Non-transcript episodes expand coverage but the pipeline shape is the same — defer until the transcript path is proven.

### Future
- **Scheduled execution**: Cron or webhook-based pipeline runs when new episodes are pushed to the repo (P2 from objective)
- **Video-to-transcript**: Integrate transcription service (Whisper, AssemblyAI, etc.) to generate transcripts for episodes that lack them
- **Multi-source ingestion**: Register additional repos beyond AI That Works
- **Topic hierarchy expansion**: Automated or semi-automated creation of child topics based on tag clustering and user interaction signals per [topic evolution strategy](Tasks/atw_nav/20 - Ideation/topic-evolution-strategy)
- **Rationale**: These are force multipliers but depend on the core pipeline being stable and the initial data proving useful.
