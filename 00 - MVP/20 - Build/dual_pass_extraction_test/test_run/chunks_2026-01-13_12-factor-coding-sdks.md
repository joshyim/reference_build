# Pass 1 & 2 Output: 2026-01-13 — Applying 12 Factor Principles to Coding Agent SDKs

**Episode**: 2026-01-13-applying-12-factor-principles-to-coding-agent-sdks
**Hosts**: Dex (HumanLayer/Riptide), Vaibhav (Boundary/BAML)
**Guest**: Mike Hostetler (Elixir/OTP agent engineering)
**Duration**: ~1h 17m
**Speakers**: 3 (Dex, Vaibhav, Mike Hostetler)

---

## Pass 1: Topic Segmentation

| Seg | Timestamps          | Topic Label                                                                | Duration |
| --- | ------------------- | -------------------------------------------------------------------------- | -------- |
| 1   | 00:01 – 01:41       | Intro banter & housekeeping                                                | ~1.5 min |
| 2   | 01:41 – 05:06       | 12-factor agents recap: tools-in-a-loop model                              | ~3.5 min |
| 3   | 05:06 – 11:53       | Consistency vs. variance tradeoff in agentic systems                       | ~7 min   |
| 4   | 11:53 – 13:20       | Aside: coding tool preferences (Claude Code, Cursor, Zed)                  | ~1.5 min |
| 5   | 13:20 – 24:00       | Control flow via prompt: why it fails, structured workflows as alternative | ~11 min  |
| 6   | 24:00 – 36:56       | Structured planning demo: design → structure → implement with schemas      | ~13 min  |
| 7   | 36:56 – 47:35       | Ralph Wiggum wrapper, deterministic harnesses, intro to Mike               | ~10 min  |
| 8   | 47:35 – 57:00       | Mike's demo: Reqit CLI, structured PRDs, Sprites.dev sandboxes             | ~10 min  |
| 9   | 57:00 – 01:06:43    | Coding workflow adoption, process personalization, team coaching           | ~10 min  |
| 10  | 01:06:43 – 01:10:10 | Q&A: SDK approach vs. manual multi-prompt, plan review UX                  | ~3.5 min |
| 11  | 01:10:10 – 01:14:18 | Vaibhav's demo: markdown-based design collaboration system                 | ~4 min   |
| 12  | 01:14:18 – 01:17:04 | Closing takeaways & next episode preview                                   | ~3 min   |

---

## Pass 2: Extracted Chunks

---

### Chunk 1: `2026-01-13-seg-02`

```yaml
chunk_id: "2026-01-13-seg-02"
episode: "2026-01-13-applying-12-factor-principles-to-coding-agent-sdks"
topic: "12-Factor Agents Recap: The Tools-in-a-Loop Model"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "01:41"
timestamp_end: "05:06"
tags: ["12-factor-agents", "agent-architecture", "tool-calling", "context-window"]
```

**Summary**: Recap of the 12-factor agents concept — replacing deterministic workflows with an agent that has a bag of tools and loops until it hits an exit condition. The promise was writing less code, but the reality was that as context windows got long (especially with mid-2024 models), models got confused and couldn't reliably follow the tool-calling loop.

**Key Insights**:
- "The idea with 12 factor agents was you could just take all of these potential state transitions and say, here's all the tools you have, here's a problem, and the model would make its way to the exit without you having to hard code all this logic."
- "The issue we found was like, as this context gets really, really long, models can get really confused and they wouldn't do a very good job."
- The reframe: move from "tools in a loop" to classifying between nodes with deterministic code in between, with small agentic sub-loops where needed.

---

### Chunk 2: `2026-01-13-seg-03`

```yaml
chunk_id: "2026-01-13-seg-03"
episode: "2026-01-13-applying-12-factor-principles-to-coding-agent-sdks"
topic: "Consistency vs. Variance Tradeoff in Agentic Systems"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "05:06"
timestamp_end: "11:53"
tags: ["reliability", "error-compounding", "agent-architecture", "consistency", "variance", "classifiers"]
```

**Summary**: Core AI engineering tradeoff. More autonomous systems handle higher input variance but lose consistency. Even 99% per-step accuracy drops to ~60% at 50 steps due to compound error. Two levers: improve per-step accuracy, or reduce number of steps. The solution is composing loops within loops — nested deterministic + agentic layers — to get both consistency and variance.

**Key Insights**:
- "You have two levers. You can make the accuracy of the tool calling better, or you can make this context window smaller, and then the poor accuracy matters less." — Dex
	- [comment] can have individual tag associated with it
- "The only two things you can do is have fewer steps or have a more accurate step selection system. Everything else is totally garbage." — Vaibhav
- Compound error math: 99% accuracy per step × 50 steps ≈ 60% overall success
- "Loops within loops within loops that compose well together — that composition is what moves it up on the stack. You're both able to increase the consistency and the variance." — Vaibhav
- Classifier pattern: small fast ML model handles 1,000 common categories; category 1,001 ("other") routes to expensive LLM. Consistency + speed for common cases, escape hatch for rare ones.

---

### Chunk 3: `2026-01-13-seg-05`

```yaml
chunk_id: "2026-01-13-seg-05"
episode: "2026-01-13-applying-12-factor-principles-to-coding-agent-sdks"
topic: "Control Flow via Prompt: Why It Fails"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "13:20"
timestamp_end: "24:00"
tags: ["prompt-engineering", "control-flow", "context-window", "planning", "structured-outputs", "instruction-following"]
```

**Summary**: Dex tells the story of embedding a 7-step workflow into a prompt — after 2 hours of adding instructions, he'd essentially rewritten a bash script. The lesson: if you know the workflow order, don't use an agent. This leads to the "Create Plan" prompt problem: a single prompt with 100+ instructions where the model skips steps (especially the high-value design discussion phase). Models can attend to ~150–200 instructions before losing track. The solution: break the monolithic prompt into separate workflow steps with structured output schemas for each.

**Key Insights**:
- "The lesson was: I had literally just written 'run these seven tools in order.' A bash script would have taken me 90 seconds." — Dex
- "Not everything is a good task for an agent. If you know the order stuff is going to happen in, you probably don't need an agent." — Dex
- "Frontier thinking LLMs can follow about 150 to 200 instructions" (as of mid-2025). Every "you MUST" / "IMPORTANT" / "CRITICAL" competes for attention.
- Context window shape matters: early back-and-forth (design discussion) is high-leverage because it steers trajectory before the model dumps tokens. Late feedback is low-leverage because most context already points one direction.
- "Editing with consistency is a much harder task than creating with consistency." — Vaibhav (on why re-steering mid-plan fails)

---

### Chunk 4: `2026-01-13-seg-06`

```yaml
chunk_id: "2026-01-13-seg-06"
episode: "2026-01-13-applying-12-factor-principles-to-coding-agent-sdks"
topic: "Structured Planning: Design → Structure → Implement with Schemas"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "24:00"
timestamp_end: "36:56"
tags: ["structured-outputs", "planning", "microagents", "workflow-design", "background-agents", "UX"]
```

**Summary**: Dex demos a structured planning workflow where each phase (design discussion, structure outline, implementation) is a separate agent call with a typed JSON schema output. The design phase outputs `{summary, current_state, desired_end_state, open_questions[], resolved_questions[]}`. Deterministic code checks "all questions answered?" to advance to the next phase. This gives smaller context windows per phase and forces compaction at natural breakpoints. Vaibhav proposes running expensive background verification agents in parallel that proactively notify users of issues.

**Key Insights**:
- Each micro-agent has an inner loop (standard Claude Code read/bash/edit) and an outer harness (structured output → deterministic transition logic).
- Design phase schema: `{summary, current_state: string[], desired_end_state: string[], open_questions: [{title, question, options[], recommendation}], resolved_questions[]}`
- "You can still take all this data and format it for the user, but you can also feed this into your deterministic code." — Dex
- BAML alternative: let the conversation be freeform, then parse the entire conversation into structured JSON at the end using a BAML function.
- Background agent pattern: "Let the user go down the golden path, but then be double-checking on their behalf with a background script doing really interesting behavior." — Vaibhav
- Judges at checkpoints don't have to be LLMs — they can be humans, manual evals, or deterministic checks. The key is having a "process-based checkpoint."

---

### Chunk 5: `2026-01-13-seg-08`

```yaml
chunk_id: "2026-01-13-seg-08"
episode: "2026-01-13-applying-12-factor-principles-to-coding-agent-sdks"
topic: "Mike's Demo: Reqit CLI — Structured Ralph Wiggum Workflows"
speakers: ["Mike Hostetler", "Dex", "Vaibhav"]
timestamp_start: "47:35"
timestamp_end: "57:00"
tags: ["ralph-wiggum", "CLI-tools", "structured-workflows", "PRD", "sprites-dev", "team-coaching"]
```

**Summary**: Mike Hostetler demos "Reqit" (Reqit Ralph), a CLI tool that wraps the Ralph Wiggum autonomous coding pattern with a deterministic structure: customized research prompts per feature → plan.md → structured PRD (prd.json with feature ID, branch, user stories with status enums). Key motivations: teaching AI engineering to a 25-person team, making prompts/outputs visible for coaching, and future integration with Sprites.dev (Fly.io's ephemeral cloud sandboxes) for parallel feature development.

**Key Insights**:
- "I wanted to strap a deterministic workflow around Ralph Wiggum." — Mike
- PRD as JSON (not markdown) enables deterministic code to orchestrate: status enums (todo/in-progress/done) let non-model code track progress.
- Template tags in prompts: each feature's research/plan/implement prompts pull structured data from prd.json.
- Coaching loop: Mike reads team members' AMP agent threads as the primary coaching tool for AI engineering skills.
- Future vision: dynamically spin up Sprites.dev sandboxes per feature, run 6 in parallel, PRs appear automatically.
- "Everyone wants a platform as a service, but the only requirement is that it has to be built in house." — Dex (on why codified workflows face adoption resistance)

---

### Chunk 6: `2026-01-13-seg-09`

```yaml
chunk_id: "2026-01-13-seg-09"
episode: "2026-01-13-applying-12-factor-principles-to-coding-agent-sdks"
topic: "Coding Workflow Adoption & Process Personalization"
speakers: ["Vaibhav", "Mike Hostetler", "Dex"]
timestamp_start: "57:00"
timestamp_end: "01:06:43"
tags: ["team-adoption", "workflow-personalization", "SDLC", "process-design"]
```

**Summary**: Discussion on whether coding agent workflows can be standardized across teams. Vaibhav observes that the more he codified his RPI workflow, the less others wanted to use it — everyone wants to customize. The industry hasn't settled on a standard process (analogous to how Agile/Jira/Linear usage varies wildly). The group discusses the spectrum of determinism: visual dependency diagrams (SVGs) for catching architectural drift, zero-warning builds to reduce context bloat, and CI/CD tooling as back pressure.

**Key Insights**:
- "The more I codified it, the less other people wanted to do it. The more Dex codified his way, the less I wanted to do it." — Vaibhav
- Vaibhav's team builds SVG dependency diagrams from their codebase: 485 lines, diffable, passable as image to agents — catches leaked dependencies visually.
- "Every build step that you run needs to run warning-free. If you're running with warnings, you will get a lot more context bloat." — Vaibhav
- Pre-commit hooks as enforcement: "no package outside of compiler packages should take dependencies on compiler packages themselves."
- Plans are too long to review effectively → the team now reviews the "structure outline" (overview) instead of the full plan.

---

### Chunk 7: `2026-01-13-seg-11`

```yaml
chunk_id: "2026-01-13-seg-11"
episode: "2026-01-13-applying-12-factor-principles-to-coding-agent-sdks"
topic: "Markdown-Based Design Collaboration System"
speakers: ["Vaibhav", "Dex", "Mike Hostetler"]
timestamp_start: "01:10:10"
timestamp_end: "01:14:18"
tags: ["design-docs", "collaboration", "markdown", "version-control", "SDLC-upstream"]
```

**Summary**: Vaibhav demos a custom markdown-based design system built by his team (replacing Notion). Features: exportable folder structures compatible with Claude Code, comment verification (AI checks whether each comment was addressed), linear versioning (not Git merging — better for design evolution), and agent-friendly editing. The key insight: design docs checkpoint decisions at a moment in time; they don't need to stay in sync with code forever.

**Key Insights**:
- "Design docs don't actually exist to help you establish your code base forever. They're to checkpoint your code base at some point in time with a context at that time." — Vaibhav
- Requirements for agent-friendly design collaboration: file as source of truth + human collaboration website + commenting engine.
- "The place where human leverage is most important shifts up to being more about the thinking and the design versus the coding bits themselves." — Dex
- Linear versioning > Git merging for design docs: "I want checkpoints that are stable and well understood and linear." — Vaibhav
- AI assistant verifies all comments are addressed → Slack notification: "all comments taken care of" or "you missed these."

---

### Chunk 8: `2026-01-13-seg-12`

```yaml
chunk_id: "2026-01-13-seg-12"
episode: "2026-01-13-applying-12-factor-principles-to-coding-agent-sdks"
topic: "Closing Takeaways"
speakers: ["Dex", "Mike Hostetler", "Vaibhav"]
timestamp_start: "01:14:18"
timestamp_end: "01:17:04"
tags: ["takeaways", "agent-architecture", "state-machines", "UX"]
```

**Summary**: Distilled recommendations from the episode.

**Key Insights**:
- "Don't use prompts for control flow. If you know what the workflow is, use control flow for control flow." — Dex
- "Start with something broad and robust for a wide range of inputs. When you learn what actual inputs look like, refine your workflow and have more happy paths. Keep the escape hatch of going fully agentic." — Dex
- "There's a place for classical AI — state machines, behavior trees. These are control flows that have been with us for 30 years. Now we're inserting this agentic loop with non-determinism and you need both." — Mike
- "Think heavily about the user's UX. Design what needs to be fast versus slow. What's synchronous? What's asynchronous? What's a background task?" — Vaibhav
