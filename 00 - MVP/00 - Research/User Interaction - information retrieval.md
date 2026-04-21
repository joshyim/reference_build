---
status: Ideation
---
# Objective

I want to allow users to look up relevant information  from the AI that Works podcast.
The main avenue will be a chat interface. the user can interact with LLM to narrow down to episodes and sections of the podcast.
I want UI to help - helper window on the side, maybe? - to select topics of interest. this will require the system to display relevant topics based on the initial conversations. when the user selects the topics, then they take priority in future retrieval

## Advantages

- **Conversational entry point is low-friction.** Users don't need to know what they're looking for upfront — they can explore naturally through dialogue, which suits a podcast library where users may not know the exact terminology or episode.
- **Topic selection as a feedback loop.** Surfacing topics and letting users pin them makes the retrieval process collaborative rather than one-shot. Gives the system a clear signal of intent without forcing the user to write a better query.
- **Narrows progressively.** The flow (open chat → see topics → select → refine) mirrors how people actually search: start broad, then focus. More natural than faceted search for unstructured audio content.

## Drawbacks / Risks

- **Topic generation quality is critical.** Out of scope — topic extraction handled in pipeline.
- **Ambiguous priority model.** Selected topics influence retrieval-phase weighting. Start with Option 1 (re-ranking) to validate UX, then evolve to Option 3 (weighted multi-query fusion) once retrieval patterns are understood. Option 1 gives a working baseline with minimal plumbing; the migration path is additive — Option 3 wraps Option 1's re-ranker into one of two search branches.
	1. **Re-ranking (post-retrieval).** Run normal search, then boost scores of results matching selected topics. Simple to implement. Risk: if relevant results didn't make the initial cut, re-ranking can't surface them.
	2. **Query expansion.** Append selected topic terms to the user's query before search. Works for both keyword and embedding. Risk: can dilute the original query if too many topics are selected.
	3. **Weighted multi-query fusion.** Run two searches — one for the chat query, one for selected topics — merge with a tunable ratio (e.g., 70/30). More complex but avoids the tradeoffs of 1 and 2.
	4. **Metadata filtering + re-rank.** If chunks are tagged with topics from the pipeline, filter to matching chunks, then rank by chat query. Depends on tagging coverage.
- **Cold start problem.** Side panel stays empty until meaningful topics emerge from conversation.
- **Dual-attention cost.** Chat window + side panel is two things to manage. Event-driven updates — panel refreshes only when the LLM detects a new topic not already shown. Topic extraction runs per-message for retrieval weighting, so the data is available without extra cost. Stale topics fade via the decay mechanism (see divergence handling below). This keeps the panel quiet unless there's something new to surface.
- **Retrieval scope.** [fixed] Removed — chunk granularity is a data prep concern, not a retrieval-phase risk. This doc assumes prepared data is already available.

## Resolved Questions

- **Underlying data:** Chunked data, searchable via keyword search and vector embeddings.
- **UI narrowing flow:** User starts broad (e.g., "I'm curious about context window"), conversation evolves to specifics (e.g., "how does 'dumb zone' affect my coding?"). Helper window surfaces domain terms like "dumb zone" as selectable items. Selecting boosts retrieval weight for that term.
- **Topic selection scope:** Single conversation only.
- **Topic divergence handling:** Decay + soft context / manual clear. Selected topics lose weight over turns as conversation drifts; user can also deselect manually. A visual indicator (e.g., dimming or subtle warning icon) appears on topics when drift is detected, giving the user a nudge without breaking flow. Decay handles the common case automatically; the visual highlight covers the edge case where the user might want to consciously decide.
