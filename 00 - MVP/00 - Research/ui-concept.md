# UI Concept: Chat + Context Sidebar

## Core Idea

Two-pane interface — chat as the primary interaction surface, persistent keyword/theme sidebar on the right as a visible, user-curated retrieval context.

## Layout

```
┌─────────────────────────────────┬──────────────────┐
│                                 │  Context Sidebar  │
│         Main Chat               │                  │
│                                 │  [Theme Tags]    │
│  User asks questions            │  [Keywords]      │
│  System responds with answers   │                  │
│  + suggests keyword tags        │  [Clear All]     │
│                                 │                  │
│  ┌─────────────────────────┐    │                  │
│  │ Input box               │    │                  │
│  └─────────────────────────┘    │                  │
└─────────────────────────────────┴──────────────────┘
```

## Interaction Loop

1. User asks a question in chat
2. System responds with an answer (with episode/code citations)
3. System also surfaces keyword/tag suggestions inline with the response
4. User clicks the keywords they find relevant
5. Selected keywords appear in the sidebar
6. Future retrieval uses conversation history + sidebar keywords as context
7. Repeat

## Sidebar Behavior

- **Sourcing**: System proposes keywords extracted from chat responses; user selects which to keep
- **Persistence**: User-selected keywords persist until explicitly removed
- **Clearing**: Individual x to remove one keyword; "Clear All" button to reset
- **Weighting**: All keywords weighted equally (v1)

## Retrieval Pipeline

The RAG retrieval step combines two context sources:

1. **Conversation context** — the current question + recent chat history
2. **Sidebar context** — all active keywords/tags selected by the user

Both feed into the retrieval query, so results stay relevant to the ongoing conversation *and* the user's declared interests.

## Tag Suggestions

> This is v2 feature. not MVP.

When the system responds to a question, it also returns a set of candidate keywords. These could be:

- Theme names (from the 8 thematic categories)
- Episode-level tags (from meta.md YAML)
- Pattern names (e.g., "backpressure", "12-factor agents")
- Technology tags (e.g., "Python", "TypeScript", "BAML")
- Content type flags (e.g., "code sample", "walkthrough", "whiteboard")

Presented inline as clickable chips/pills below each chat response.

## Example Chat Response

> **User**: How do people handle context windows getting too large?
>
> **System**: Episodes #14 and #31 both address context management strategies.
> Episode #14 introduces a "sliding window + summary" pattern where older context
> gets compressed into a summary block. Episode #31 builds on this with a
> priority-based approach — tagging context chunks by relevance and dropping
> low-priority ones first. Code samples in both episodes (Python).
>
> **Suggested tags**: `context engineering` · `context window` · `summarization` · `Python`

The suggested tags are shown in the sidebar (v2). 
User can select or decline the tags. User selections is used to refine the query for future responses.

## Open Design Questions

- How many tags suggestions per response? (3–5 feels right)
- Should the sidebar group keywords by type (themes vs. technologies vs. patterns)?
- Should there be a "suggested starting keywords" state for new sessions?
- Mobile/narrow viewport — does the sidebar collapse into a drawer?
- Should clicking a sidebar keyword show which episodes/insights match it?
