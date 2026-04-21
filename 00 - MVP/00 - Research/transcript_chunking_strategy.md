# Transcript Chunking Strategy — AI That Works Episodes

## Transcript Format Analysis

The transcripts follow a consistent format across episodes. Here's what we're working with:

### Structure

```
Speaker Name (HH:MM:SS.ms)
Paragraph of spoken text.

Speaker Name (HH:MM:SS.ms)
Next spoken text.
```

- **No headers, no frontmatter, no markdown structure** — just raw speaker turns separated by blank lines
- Each turn = speaker label with timestamp + one or more lines of text
- Timestamps are wall-clock from the recording (e.g., `00:01.207` to `01:17:04.131`)
- Some turns are very short (1-2 words, e.g., "Yeah." or "You") — these are crosstalk/backchannels

### Quantitative Profile (from `2026-01-13` episode)

| Metric | Value |
|---|---|
| Total lines | ~1,118 |
| Total characters | ~82K |
| Speaker turns | ~311 |
| Unique speakers | 3 (Dex, Vaibhav, Mike Hostetler) |
| Duration | ~1h 17m |
| Blank lines (turn separators) | ~403 |

### Availability

Transcripts exist for a subset of episodes (confirmed: 7 out of 53+). All confirmed transcripts are from late 2025 onward:
- `2025-12-02-multimodal-evals`
- `2025-12-09-git-worktrees` (~765 lines)
- `2025-12-16-prompt-optimizer`
- `2025-12-23-founding-humanlayer`
- `2025-12-30-founding-boundary`
- `2026-01-06-latency`
- `2026-01-13-applying-12-factor-principles-to-coding-agent-sdks` (~1,118 lines)

[comment] Not all episodes have transcripts. The episodes that don't have transcript.md files will need alternative content sources (README.md, meta.md, whiteboards.md, journal.md, walkthroughs).

---

## Chunking Challenges

1. **No topic structure** — transcripts are flat streams of conversation. There are no headers, section breaks, or topic markers. The speakers don't announce "now we're talking about X."
2. **Noisy content** — Lots of filler ("Yeah", "Mm-hmm", "Go ahead"), crosstalk, tangents, audience interaction, tool/screen sharing logistics.
3. **Long-range coherence** — A single concept (e.g., "consistency vs. variance tradeoff") can span 20+ speaker turns across 5 minutes, with interruptions mixed in.
4. **Multi-topic episodes** — Episodes typically cover 3-5 distinct topics/demos, plus intro banter and closing takeaways.

---

## Proposed Chunking Strategy

### Approach: Two-Pass Semantic Chunking

Rather than mechanical splitting (e.g., every N tokens), use a two-pass approach that respects the conversation's natural structure.

---

### Pass 1: Segmentation (Break into topical segments)

**Method**: Use an LLM to identify topic boundaries.

**Input**: The full transcript (or sliding window if it exceeds context).

**Prompt pattern**:
> Given this podcast transcript, identify the major topic segments. For each segment, provide:
> - Start timestamp
> - End timestamp
> - Topic label (short descriptive name)
> - 1-sentence summary
>
> Ignore: intro banter, logistics (screen sharing, Discord), and brief tangents under 30 seconds.

**Expected output** for the `2026-01-13` episode would be something like:

| Segment | Timestamps | Topic |
|---|---|---|
| 1 | 01:41 – 05:06 | Intro: 12-factor agents recap and episode framing |
| 2 | 05:06 – 11:53 | Consistency vs. variance in agentic workflows |
| 3 | 11:53 – 13:06 | Aside: coding tools comparison (Claude Code, Cursor) |
| 4 | 13:06 – ~30:00 | Structured workflows: research → plan → implement |
| 5 | ~30:00 – ~55:00 | Mike's demo: Reqit (structured CLI for agentic workflows) |
| 6 | ~55:00 – ~72:00 | Vaibhav's demo: markdown-based design collaboration system |
| 7 | ~72:00 – 77:04 | Closing takeaways and recommendations |

This gives us **5-8 chunks per episode**, each representing a coherent topic block.

---

### Pass 2: Extraction (Turn segments into useful chunks)

For each segment from Pass 1, extract a structured chunk:

```yaml
chunk_id: "2026-01-13-seg-02"
episode: "2026-01-13-applying-12-factor-principles-to-coding-agent-sdks"
episode_title: "Applying 12 Factor Principles to Coding Agent SDKs"
topic: "Consistency vs. Variance in Agentic Workflows"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "05:06"
timestamp_end: "11:53"
summary: >
  Discussion of the fundamental tradeoff in AI engineering between
  consistency (reliable outputs) and variance (handling diverse inputs).
  As autonomy increases, variance capacity goes up but consistency drops.
  Two levers: improve accuracy of each step, or reduce number of steps.
key_insights:
  - "You have two levers: make accuracy better or make the context window smaller"
  - Compound error rate: even 99% per-step accuracy drops to ~60% at 50 steps
  - Classical state machines + selective agentic loops = best of both worlds
tags: ["agent-architecture", "reliability", "error-compounding", "12-factor-agents"]
raw_text: |
  <the cleaned transcript text for this segment>
```

---

### Cleaning Rules (applied during Pass 2)

1. **Drop backchannels** — Remove turns that are only filler ("Yeah.", "Mm-hmm.", "Go ahead.")
2. **Merge split turns** — When the same speaker has consecutive turns within 5 seconds, merge them
3. **Strip logistics** — Remove screen-sharing narration, Discord mentions, "can you see my screen" etc.
4. **Preserve quotes** — Keep verbatim phrasing for key insights (these become the `key_insights` field)
5. **Attribute speaker** — Maintain speaker labels so we know who said what

---

### Chunk Sizing Guidelines

| Parameter | Value | Rationale |
|---|---|---|
| Target chunk size | 1,000–3,000 tokens | Fits RAG retrieval windows; contains enough context for a complete idea |
| Min chunk size | 500 tokens | Shorter segments get merged with adjacent content |
| Max chunk size | 5,000 tokens | Longer segments get split at natural sub-topic breaks |
| Overlap | Include last 2 turns of prior segment as context | Maintains conversational coherence at boundaries |

---

## Alternative: Simpler Mechanical Approach

If LLM-based segmentation is too expensive/slow for the pipeline, a fallback:

1. **Split by time windows** — Group turns into 5-minute windows
2. **Merge short turns** — Combine consecutive same-speaker turns
3. **Drop noise** — Filter turns under 10 words
4. **Summarize each window** with an LLM in a single pass

This is cheaper but produces lower-quality chunks because 5-minute windows will split mid-topic.

---

## Integration with Existing Data Sources

For the majority of episodes that **don't have transcripts**, we still have:
- `meta.md` — YAML metadata with topics, description, video URL
- `README.md` — episode description and code walkthrough
- `whiteboards.md` — conceptual diagrams as text
- `journal.md` — session notes
- `walkthrough.md` — step-by-step guides

[suggestion] Consider building a unified chunk schema that works for both transcript-derived chunks and non-transcript chunks. The schema above works for both — just populate `raw_text` from the available source and adjust `timestamp_start`/`timestamp_end` to be optional.

---

## Implementation Sequence

1. **Start with the 7 episodes that have transcripts** — these are the richest source material
2. Run Pass 1 (segmentation) on each transcript
3. Run Pass 2 (extraction) to produce structured chunks
4. For remaining episodes, generate chunks from README + meta.md + available markdown files
5. Merge all chunks into the thematic index described in `basic_research.md`

---

## Open Questions

- [ ] What embedding model / vector store (if any) for RAG, or is this purely for generating the static Markdown navigator?
- [ ] Should we preserve the full cleaned transcript text in chunks, or just the summary + key quotes?
- [ ] Worth transcribing episodes that don't have transcript.md? (YouTube videos exist for most)
