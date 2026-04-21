# Topic Evolution Strategy

## Problem

Insights extracted from episodes are inherently non-comprehensive. An episode on "evals" might only cover programmatic evals — not benchmarks, LLM-as-Judge, ROUGE, SWEBench, etc. The topic taxonomy can't be defined upfront because the field is evolving and new concepts emerge constantly.

The topic structure needs to grow organically from how users actually interact with the content, not from the raw episode data alone.

## Two-Layer Design

| Layer | Entity | Nature | Source |
|---|---|---|---|
| Informal | **Keyword** | Free-form, ephemeral | LLM-generated per chat response, user-selected |
| Structured | **Topic** | Hierarchical, curated | Promoted from keywords based on usage signal |

Keywords are the sensing layer. Topics are the knowledge layer. Keywords graduate into topics.

## Evolution Lifecycle

### Phase 1: Keyword Accumulation

Every chat response suggests 3–5 keyword chips. Users click the ones they find relevant. These selections are stored in `session_keyword`.

Over time, patterns emerge:
- "LLM-as-Judge" keeps getting selected across different sessions
- "programmatic evals" clusters with "eval frameworks"
- Users ask questions that existing topics don't cover well

### Phase 2: Signal Detection

Aggregate keyword selection data to identify candidates for promotion:
- **Selection frequency** — how often is this keyword selected?
- **Session spread** — is it one user or many?
- **Recency** — is it trending or stale?
- **Retrieval gaps** — are queries using this keyword returning low-confidence results?

This can be a materialized view over `session_keyword` and `message_keyword`, or a lightweight `keyword_signal` summary:

| Field | Type | Notes |
|---|---|---|
| `keyword_id` | string | |
| `selection_count` | int | Total selections |
| `distinct_sessions` | int | Breadth of interest |
| `first_seen` | timestamp | |
| `last_seen` | timestamp | |

### Phase 3: Keyword → Topic Promotion

When a keyword reaches sufficient traction, it gets promoted:

1. **Create a new topic** — new row in `topic` table with appropriate `parent_id` (e.g., "LLM-as-Judge" under "Evals")
2. **Link the keyword** — set `keyword.linked_topic_id` to the new topic
3. **Backfill insight links** — review existing insights and create `insight_topic` rows with appropriate `coverage` values

Example: The "Evals" topic tree grows from:
```
Evals
```
to:
```
Evals
├── Programmatic Evals
├── LLM-as-Judge
├── Benchmarks (SWEBench, Tau Bench)
└── Qualitative Output Evaluation
```

Each child topic only gets created when user interaction signals demand for it.

### Phase 4: Insight Backfill

When a new topic is created, existing insights may need to be re-linked:

- The original evals episode insight gets: `insight_topic(topic=programmatic-evals, coverage=deep)`
- That same insight might also get: `insight_topic(topic=llm-as-judge, coverage=tangential)` if it briefly mentioned the concept
- Older insights that were only linked to the parent "Evals" topic can be re-evaluated against the new children

This backfill can be manual (curator review) or assisted (LLM re-reads the insight text and suggests topic links).

### Phase 5: Ongoing Reinforcement

As new episodes drop and new insights are created:
- New insights get tagged to existing topics where they fit
- If an insight doesn't fit well into any existing topic, its keywords become signal for future topics
- The cycle repeats

## Key Principle

**The topic tree is never "complete."** It's a living taxonomy that reflects what users actually care about, not an upfront classification scheme. The data model supports this by keeping topics as a simple table with recursive hierarchy — no schema changes needed to grow it.

## Open Questions

- **Who promotes keywords to topics?** Options: manual curation, automated threshold, LLM-assisted with human approval
- **How aggressive should backfill be?** Re-evaluate all insights, or only insights from episodes tagged with related `source_tags`?
- **Should promoted topics inherit the keyword's selection history?** Useful for showing "trending" topics
- **Merge/split:** What happens when two topics turn out to be the same thing, or one topic needs to split?
