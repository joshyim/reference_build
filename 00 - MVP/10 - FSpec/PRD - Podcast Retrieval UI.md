---
status: Draft
source:
  - "Tasks/atw_nav/10 - Direction Setting/User Interaction - information retrieval.md"
  - "Tasks/atw_nav/20 - Ideation/frontend/ui-concept.md"
---

# PRD - Podcast Retrieval UI

## 1. Executive Summary

Enable users to find relevant information from the AI that Works podcast through a conversational chat interface paired with a tag selection sidebar. The primary outcome is low-friction discovery — users start with a vague question and progressively narrow to specific episodes and segments without needing to know exact terminology upfront. The system uses keyword and vector search over pre-chunked transcript data, with tag-based re-ranking that evolves based on user selections and conversational drift.

## 2. Product Overview

### Purpose
A two-pane retrieval interface — chat as the primary interaction surface, with a persistent context sidebar on the right as a visible, user-curated retrieval context. Users chat with an LLM that searches chunked transcript data and surfaces results. The companion sidebar displays relevant tags extracted from the conversation, which users can select to steer retrieval weighting.

### Layout

```
┌─────────────────────────────────┬──────────────────┐
│                                 │  Context Sidebar  │
│         Main Chat               │                  │
│                                 │  [Theme Tags]    │
│  User asks questions            │  [Tags]          │
│  System responds with answers   │                  │
│  + suggests tags                │  [Clear All]     │
│                                 │                  │
│  ┌─────────────────────────┐    │                  │
│  │ Input box               │    │                  │
│  └─────────────────────────┘    │                  │
└─────────────────────────────────┴──────────────────┘
```

### Key Features
- **Chat-based search:** Users ask natural language questions; the system retrieves and presents relevant podcast segments with episode/code citations.
- **Context sidebar:** Surfaces domain-specific tags and concepts as the conversation evolves. Users select tags to influence retrieval weighting.
- **Progressive narrowing:** The flow moves from broad exploration to specific segments through dialogue and tag refinement.
- **Decay and drift handling:** Selected tags lose weight over turns if the conversation moves on, with visual indicators on stale tags.

### Core Interaction Loop
1. User asks a question in chat
2. System responds with an answer (with episode/code citations)
3. System surfaces tag suggestions inline with the response
4. User clicks the tags they find relevant
5. Selected tags appear in the sidebar
6. Future retrieval uses conversation history + sidebar tags as context
7. Repeat

### Strategic Alignment
Extends the ATW Nav platform by making podcast content accessible through conversational interaction rather than requiring users to browse episodes or search transcripts manually.

## 3. Target Persona

### Listener
- **Pain points:** Remembers hearing something interesting but can't recall the episode or exact phrasing; wants to explore a specific concept across multiple episodes; may be unfamiliar with the podcast's vocabulary.
- **Need:** A conversational way to search podcast content that works whether they know the exact terminology or not, and that supports both quick lookups and deeper exploration.
- **Motivation:** Find relevant podcast segments without re-listening to full episodes or manually browsing the catalog.

## 4. Functional Requirements (Epics)

### Epic 1: Chat-Based Retrieval

> As a **user**, I want to ask natural language questions about the podcast and get relevant segments back, so that I can find information without knowing exact episode titles or timestamps.

**In Scope:**
- User submits a text query via chat interface
- System performs keyword search and vector embedding search against chunked transcript data
- Results are ranked and presented in the chat as segment excerpts with episode references
- Conversation context from prior turns informs subsequent retrieval (multi-turn)

**Out of Scope:**
- Audio playback from within the chat interface
- Real-time transcription or live episode support
- Query autocomplete or suggested queries

### Epic 2: Context Sidebar (v2)

> As a **user**, I want to see relevant tags surfaced from my conversation, so that I can select them to focus the search on what I care about.

**In Scope:**
- Sidebar displays tags extracted from the conversation by the LLM
- **Sourcing:** System proposes tags extracted from chat responses; user selects which to keep
- **Persistence:** User-selected tags persist within the session until explicitly removed
- **Clearing:** Individual ✕ to remove one tag; "Clear All" button to reset
- **Weighting:** All tags weighted equally (v1)
- Panel updates are event-driven — refreshes only when a new tag is detected, not on every turn
- Panel is empty at conversation start (cold start); appears once meaningful tags emerge
- Tag selection scope is per-conversation only (does not persist across sessions)

**Out of Scope:**
- User-defined custom tags (freeform tag entry)
- Tag persistence across conversations
- Tag taxonomy management or editing
- Tag grouping by type (themes vs. technologies vs. patterns) — revisit after v1 usage data

### Epic 3: Tag-Weighted Re-Ranking (v2)

> As a **user**, I want my selected tags to influence which results appear first, so that the system prioritizes what I've told it I care about.

**In Scope:**
- Retrieval combines two context sources:<br>  1. **Conversation context** — the current question + recent chat history<br>  2. **Sidebar context** — all active tags selected by the user
- Both feed into the retrieval query, so results stay relevant to the ongoing conversation *and* the user's declared interests
- Selected tags boost scores of matching results in a post-retrieval re-ranking step
- Re-ranking applies after standard keyword + vector search completes
- Multiple selected tags contribute additively to score boosting

**Out of Scope:**
- Weighted multi-query fusion (Phase 2 — Option 3)
- Tag weighting configuration by the user (e.g., sliders)
- Filtering results exclusively to selected tags (tags are preferences, not hard filters)

### Epic 4: Tag Decay and Drift Handling (v2)

> As a **user**, I want stale tag selections to fade naturally as my conversation shifts, so that I don't have to manually manage my selections as I explore.

**In Scope:**
- Selected tags lose retrieval weight over turns as conversation drifts away from them
- Visual indicator (dimming or subtle warning icon) appears on tags when drift is detected
- User can manually deselect tags at any time
- Tags that are re-engaged in conversation regain weight

**Out of Scope:**
- Automatic tag removal (decay reduces weight but doesn't delete selections)
- Prompting the user with explicit "clear stale tags?" dialogs
- Configurable decay rates

### Epic 5: Inline Tag Suggestions (v2)

> As a **user**, I want to see suggested tags below each chat response, so that I can quickly add relevant terms to my sidebar context without typing.

**In Scope:**
- System returns candidate tags inline as clickable chips/pills below each chat response
- Tag types include: theme names, episode-level tags, pattern names (e.g., "backpressure", "12-factor agents"), technology tags (e.g., "Python", "TypeScript", "BAML"), content type flags (e.g., "code sample", "walkthrough", "whiteboard")
- 3–5 tag suggestions per response
- Clicking a tag adds it to the sidebar; declining leaves it inactive

**Out of Scope (v2):**
- Suggested starting tags for new sessions
- Tag suggestion ranking/ordering logic
- Tag suggestion deduplication across responses

### Cross-Cutting Out of Scope
- Data pipeline and transcript chunking (handled separately — this spec assumes prepared data is available)
- Tag extraction model/pipeline (out of scope — handled in data prep)
- User authentication or personalization across sessions
- Analytics dashboard for retrieval performance
- Mobile-specific UI considerations (sidebar collapse/drawer behavior — revisit for responsive design pass)

## 5. User Stories

### Epic 1: Chat-Based Retrieval

| Title                           | Story Detail                                                                                                                                                                              | Acceptance Criteria                                                                                                                                                                                                                                                                                     | Priority |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| Ask a question in chat          | As a **Listener**, I want to type a natural language question into a chat input and submit it, so that I can search for something without knowing episode titles or timestamps.           | - Chat interface displays a text input field with a send button<br>- User can submit via Enter key or send button<br>- Submitted query appears as a user message bubble in the chat<br>- Empty or whitespace-only submissions are blocked with inline feedback                                          | High     |
| Retrieve matching segments      | As a **Listener**, I want the system to return relevant transcript segments in response to my question, so that I can read the most pertinent content without listening to full episodes. | - System returns one or more segment excerpts in the chat response<br>- Each excerpt preserves enough surrounding context to be comprehensible standalone<br>- Results are ordered by relevance score (highest first)<br>- If no relevant segments are found, a clear "no results" message is displayed | High     |
| Multi-turn context carry        | As a **Listener**, I want follow-up questions understood in the context of my prior turns, so that I can progressively narrow my search without repeating background each time.           | - Prior conversation turns are included as retrieval context for subsequent queries<br>- Anaphoric references ("tell me more about that", "what else did they say") resolve against prior context<br>- Context window is bounded to recent turns to maintain relevance                                  | High     |
| Episode source citations        | As a **Listener**, I want each returned segment to display a clear episode reference, so that I know where to go if I want to listen to the full episode.                                 | - Each segment excerpt includes an episode identifier (title and/or code)<br>- Citations are visually distinct from excerpt body text<br>- Citation format is consistent across all results in a single response                                                                                        | High     |
| Graceful empty-state experience | As a **Listener**, I want to see a welcoming empty state when I first open the chat, so that I understand what I can do and feel encouraged to ask my first question.                     | - Chat pane displays a brief description of what the tool does before any messages<br>- One or two example queries are shown as clickable prompts<br>- Empty state disappears once the user submits their first query                                                                                   | Medium   |


**Out of scope for this epic:** Audio playback from within the chat interface, real-time transcription or live episode support, query autocomplete or suggested queries.

### Epics 2–5: Deferred

User stories for Context Sidebar, Tag-Weighted Re-Ranking, Tag Decay and Drift Handling, and Inline Tag Suggestions will be defined when those epics are prioritized for development.

## 6. Success Metrics

### Leading Indicators (change quickly)
- **Query-to-result click-through rate** — % of search results that users engage with → target: >40%
- **Tag selection rate** — % of conversations where users select at least one tag → target: >30%
- **Turns to first useful result** — average number of chat turns before the user finds a relevant segment → target: ≤3 turns

### Lagging Indicators (change over time)
- **Return usage rate** — % of users who use the retrieval interface more than once per week → target: >25%
- **Session depth** — average number of turns per conversation → target: >5 (indicates exploration, not frustration)
- **Sidebar engagement over time** — trend of tag selection rate across cohorts → target: increasing month-over-month

## 7. Open Questions

- **[Design]** What is the exact visual treatment for decayed/drifted tags? Dimming, icon, opacity change, or combination?
- **[Design]** How many tag suggestions per response? (3–5 feels right, needs validation)
- **[Design]** Should the sidebar group tags by type (themes vs. technologies vs. patterns)?
- **[Design]** Should there be a "suggested starting tags" state for new sessions?
- **[Design]** Mobile/narrow viewport — does the sidebar collapse into a drawer or bottom sheet?
- **[Design]** Should clicking a sidebar tag show which episodes/insights match it?
- **[Engineering]** What decay function to use? Linear, exponential, or step-based? What's the half-life in turns?
- **[Engineering]** How does re-ranking score boosting interact with the existing keyword + vector score combination? Additive? Multiplicative? Separate pass?
- **[Data]** What tag granularity does the extraction pipeline produce? Are tags single terms, phrases, or hierarchical?

## 8. Roadmap

### Now
- **Chat-based retrieval** (Epic 1) — core value proposition; everything else depends on this working well.
- **Context sidebar** (Epic 2) — tag display with select/deselect, individual remove, and "Clear All".
- **Tag-weighted re-ranking** (Epic 3, Phase 1) — post-retrieval score boosting from selected tags.

**Rationale:** These three epics deliver the full user-facing loop (ask → see tags → select → get better results) with the simplest retrieval approach (Option 1 re-ranking). Validates whether the sidebar adds value before investing in more complex retrieval or inline suggestions.

### Next
- **Tag decay and drift handling** (Epic 4) — decay weighting, visual drift indicators, manual clear.
- **Inline tag suggestions** (Epic 5) — clickable tag chips below chat responses.
- **Weighted multi-query fusion** (Phase 2 of Epic 3) — evolve from re-ranking to dual-search with tunable ratio.

**Rationale:** Decay only matters once users are actually selecting tags and having multi-turn conversations — needs real usage data to tune. Inline tag suggestions (v2 feature) depend on the sidebar working well first. Multi-query fusion is an optimization on retrieval quality that builds on the re-ranking baseline.

### Future
- Cross-session tag memory (persistent preferences)
- Audio playback integration (jump to timestamp from result)
- Retrieval analytics and feedback loop (thumbs up/down on results to improve ranking)
- Responsive design pass for mobile/narrow viewports (sidebar → drawer/bottom sheet)

**Rationale:** These extend the core experience but require infrastructure (user accounts, audio player, feedback pipeline) that isn't justified until the base retrieval + tag flow is validated.
