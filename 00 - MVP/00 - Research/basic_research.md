# Research: AI That Works — Insight Navigator

## Problem

The [AI That Works](https://github.com/ai-that-works/ai-that-works) repo is a 53+ episode archive from a weekly live-coding show on production AI engineering. It contains code samples, markdown guides, whiteboard notes, walkthroughs, and video links — but everything is organized chronologically by episode date. There's no way to quickly find insights by topic, pattern, or use case.

**Source repo**: [https://github.com/ai-that-works/](https://github.com/ai-that-works/ai-that-works)

## What's In The Repo

| Content Type | Where It Lives | Volume |
|---|---|---|
| Episode folders | `YYYY-MM-DD-episode-name/` | ~53 episodes |
| Episode metadata | `meta.md` per episode (YAML frontmatter) | Per episode |
| Distilled guide | `HOWTO.md` (28KB) | 1 file, covers all key themes |
| Structured data | `data.json`, `feed.xml` | Auto-generated from metadata |
| Code samples | `final/`, `src/`, `hack/` subdirs | TypeScript, Python, Go, BAML |
| Walkthroughs | `step-by-step/walkthrough.md` | Select episodes |
| Whiteboard notes | `whiteboards.md` | Select episodes |
| Session journals | `journal.md` | Select episodes |

### Content Categories

- **Regular episodes** (#1–#48): Weekly live-coding sessions
- **Workshops**: Multi-day deep dives (NYC, SF — Twelve Factor Agents)
- **"No Vibes Allowed"**: Hands-on coding-focused episodes
- **Founding stories**: Business/product lessons from founders (HumanLayer, Boundary)

### Key Topics Covered

| Time Period | Topics |
|---|---|
| Early 2025 | Classification, reasoning models, code generation, 12-factor agents |
| Mid 2025 | MCP tools, evals design, entity extraction, content pipelines, context engineering |
| Late 2025 | Memory systems, multimodality, vision, interruptible agents, event-driven architecture, voice agents |
| End 2025 | Prompt optimization, git worktrees, founding stories |
| 2026 | Latency optimization, SDK principles, email agents, backpressure, PII redaction, agent skills |

## Goal

Build a navigable insight index that lets someone quickly find what they need by **topic** rather than by **episode date**.

## Proposed Approach

### 1. Thematic Index

Group insights into navigable themes:

- **Context Engineering** — prompt design, context window management, structured outputs
- **Agent Architecture** — 12-factor agents, state management, tool orchestration
- **Evaluation & Testing** — evals frameworks, testing strategies, quality measurement
- **MCP & Tool Orchestration** — Model Control Protocol, tool integration, composability
- **Data & Extraction** — entity extraction, data pipelines, PII handling
- **Voice & Multimodal** — voice agents, vision models, multimodal patterns
- **Production Patterns** — deployment, monitoring, latency, backpressure
- **Founding Stories** — business and product lessons from founders

### 2. Per-Insight Cards

Each insight entry should include:
- Source episode (number, title, date)
- Video link (from `meta.md`)
- Key takeaway (1–2 sentences)
- Reusable pattern name (if applicable)
- Code pointer (repo-relative path + what it demonstrates)
- Cross-references to related episodes

### 3. Learning Paths

Per theme, suggest an episode order for progressive learning (e.g., "Start with Episode #5 for foundations, then #12 for advanced patterns").

### 4. Quick-Reference Indexes

- **Episode index** (chronological table with theme tags, code/walkthrough flags)
- **Code sample index** (pattern → language → episode → file path)
- **Top principles** distilled from `HOWTO.md` with episode citations

## Existing Navigation (What Already Exists)

The repo already provides:
- `README.md` — auto-generated episode table (chronological, links to videos)
- `data.json` — machine-readable episode metadata
- `feed.xml` — RSS feed
- `HOWTO.md` — distilled best practices (not linked to specific episodes)
- `meta.md` per episode — YAML metadata with topics, descriptions, video URLs

These are useful inputs but don't solve topical discovery.

## Output

A single Markdown document (or small set of documents) that serves as the insight navigator. Should be usable in Obsidian.

## Open Questions

- [ ] Should this be a one-time generated document, or a repeatable process that updates as new episodes drop?
- [ ] Is a single large file preferred, or one file per theme?
- [ ] Should we include full code snippets inline, or just pointers to repo paths?
- [ ] Any topics to prioritize or deprioritize?
- [ ] Is there value in a searchable format beyond Markdown (e.g., a local web page, or structured JSON)?
