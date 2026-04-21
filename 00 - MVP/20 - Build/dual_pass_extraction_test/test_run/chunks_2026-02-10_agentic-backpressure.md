# Pass 1 & 2 Output: 2026-02-10 — Agentic Backpressure Deep Dive

**Episode**: 2026-02-10-agentic-backpressure-deep-dive
**Hosts**: Dex (HumanLayer/Riptide), Vaibhav (Boundary/BAML)
**Duration**: ~56 min
**Speakers**: 2 (Dex, Vaibhav)

---

## Pass 1: Topic Segmentation

| Seg | Timestamps | Topic Label | Duration |
|-----|-----------|-------------|----------|
| 1 | 00:00 – 01:42 | Stream start chaos & intro | ~2 min |
| 2 | 01:42 – 03:32 | Announcements: schedule updates, 50th episode unconference | ~2 min |
| 3 | 03:32 – 08:30 | Vaibhav's wrong-assumption story: BAML type system design | ~5 min |
| 4 | 08:30 – 14:23 | Dex's wrong-assumption story: Claude Agent SDK behavior; intro to learning tests | ~6 min |
| 5 | 14:23 – 20:30 | Learning tests defined: Michael Feathers, proof-based development, evals as learning tests | ~6 min |
| 6 | 20:30 – 27:36 | Learning tests in practice: hello world → assertions → unit test frameworks | ~7 min |
| 7 | 27:36 – 35:10 | Real-world parallels: database prechecks, fuzz testing, when to apply learning tests vs. design | ~7.5 min |
| 8 | 35:10 – 39:10 | Developing instinct: too-much-planning vs. not-enough, binary search for ideal range | ~4 min |
| 9 | 39:10 – 41:34 | Live demo: Claude writes & runs learning test for new Stream Send API | ~2.5 min |
| 10 | 41:34 – 46:07 | Backpressure defined: compiler, type system, tests, MCP as automated feedback loops | ~4.5 min |
| 11 | 46:07 – 50:26 | Vaibhav's dependency diagram demo: SVG visualization, CI/CD enforcement, zero-warning builds | ~4.5 min |
| 12 | 50:26 – 53:12 | LLM-as-judge critique: deterministic backpressure > model-based judging | ~3 min |
| 13 | 53:12 – 56:01 | Closing: developing instinct, professional craft, next episode preview | ~3 min |

---

## Pass 2: Extracted Chunks

---

### Chunk 1: `2026-02-10-seg-03`

```yaml
chunk_id: "2026-02-10-seg-03"
episode: "2026-02-10-agentic-backpressure-deep-dive"
topic: "Wrong Assumptions in Complex Design: BAML Type System Story"
speakers: ["Vaibhav", "Dex"]
timestamp_start: "03:32"
timestamp_end: "08:30"
tags: ["design-assumptions", "type-systems", "architecture", "BAML"]
```

**Summary**: Vaibhav describes hitting a fundamental design assumption error at 2:45 AM. BAML has three type systems (compiler, streaming, non-streaming) that each need the same type simplification algorithm implemented slightly differently. The architectural problem: the same algorithm three times with different semantics. This is a design-level mistake that no learning test can catch — it requires design rethinking. Sets up the contrast with implementation-level assumptions that learning tests *can* catch.

**Key Insights**:
- Three parallel type systems with "almost exactly the same thing implemented three times, but they all have totally different semantics" — a design philosophy problem, not an implementation bug.
- Distinction established: design-level wrong assumptions need design rethinking; implementation-level wrong assumptions need learning tests.
- "There's no learning test I can do there. That just requires design." — Vaibhav

---

### Chunk 2: `2026-02-10-seg-04`

```yaml
chunk_id: "2026-02-10-seg-04"
episode: "2026-02-10-agentic-backpressure-deep-dive"
topic: "Wrong Assumptions with External APIs: Claude Agent SDK"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "08:30"
timestamp_end: "14:23"
tags: ["external-APIs", "black-box-testing", "learning-tests", "Claude-Agent-SDK", "research"]
```

**Summary**: Dex's parallel story: building on the Claude Agent SDK, which wraps a closed-source CLI. Standard research-plan-implement relies on reading code, but you can't read the code of external dependencies that call closed-source binaries. Assumptions about `allowedTools` behavior leaked through research → plan → implementation, causing late-stage rework. The answer: read the docs (pull them into research), but docs aren't enough — you also need to actually run the system. Introduces Michael Feathers' "learning tests" concept.

**Key Insights**:
- "Most of the code here is a system we don't control, and we can't read the code." — Dex (on SDK wrapping a closed-source binary)
- Standard research pipeline assumes "we can get all the knowledge we need to correctly build this feature by reading the code." This breaks with external/closed-source dependencies.
- Three-layer research: (1) read your code, (2) read external docs/blog posts, (3) write learning tests.
- "We're not writing code to ship a feature, we're writing code to prove the system works the way we think it does." — Dex

---

### Chunk 3: `2026-02-10-seg-05`

```yaml
chunk_id: "2026-02-10-seg-05"
episode: "2026-02-10-agentic-backpressure-deep-dive"
topic: "Learning Tests Defined: Proof-Based Development"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "14:23"
timestamp_end: "20:30"
tags: ["learning-tests", "proof-based-development", "evals", "research-pipeline", "Michael-Feathers"]
```

**Summary**: Formal definition of learning tests (from Michael Feathers' "Working Effectively with Legacy Code"): writing tests against external systems you don't control to document how they actually behave. Not for shipping code — for understanding behavior. Evals are a form of learning tests ("if I put this prompt in, how will the LLM behave?"). The goal is to vet assumptions *before* proceeding to implementation, catching wrong assumptions at the highest-leverage point in the pipeline.

**Key Insights**:
- "Learning tests" — term from Michael Feathers: "Systems that are hard to understand, maybe you just jump in and poke them from the outside."
- "The best learning test is actually when you're calling the LLM. The only way to evaluate..." — Vaibhav → evals = learning tests for LLM behavior.
- Performance engineering analogy: "You don't model the assembly. You literally write the code, you look at the assembly that gets generated, and you're like, cool, this is the slot I want to reduce." — Vaibhav
- "Print debugging has overtaken GDB debugging because it's just a learning test. That's what you're doing." — Vaibhav
- Leverage argument: wrong assumption in research → leaks into plan → leaks into implementation → rework at phase 3 wastes everything before it.

---

### Chunk 4: `2026-02-10-seg-06`

```yaml
chunk_id: "2026-02-10-seg-06"
episode: "2026-02-10-agentic-backpressure-deep-dive"
topic: "Learning Tests in Practice: From Hello World to Contract Testing"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "20:30"
timestamp_end: "27:36"
tags: ["learning-tests", "contract-testing", "external-dependencies", "SDK-testing", "Claude-Agent-SDK"]
```

**Summary**: Progressive examples of learning tests: (1) Hello world — just run the SDK and print output to see message structure. (2) Add boolean flags and observations — "did we get a session ID?" (3) Put assertions into a unit test framework — document expected behavior. (4) Contract testing — when the external library updates (e.g., SDK v1 → v2 changed session continuation behavior), rerun learning tests to see what broke. The team has ~100 learning tests documenting their contract with external systems.

**Key Insights**:
- "You wouldn't run these all the time, but you have a little bit of a demonstration. When you want to write code with this library, the model can go read this really useful reference." — Dex
- Contract change detection: "Claude SDK 1 to SDK 2, they changed the default behavior where now you have to pass in `forkSession: true`." Learning test caught it.
- "Every time a contract with our external system breaks, we add another learning test." — Dex
- "You're treating something like a probabilistic system. It has some probability of producing something. You're trying to constrain the probabilities." — Vaibhav
- "Unit tests for external fuzzy libraries" — Vaibhav's summary of the pattern.

---

### Chunk 5: `2026-02-10-seg-07`

```yaml
chunk_id: "2026-02-10-seg-07"
episode: "2026-02-10-agentic-backpressure-deep-dive"
topic: "When to Apply Learning Tests vs. Just Ship"
speakers: ["Vaibhav", "Dex"]
timestamp_start: "27:36"
timestamp_end: "35:10"
tags: ["learning-tests", "tradeoffs", "decision-making", "research-pipeline"]
```

**Summary**: The hardest part isn't implementing learning tests — it's knowing when to apply them vs. just one-shotting. If you do them too early on simple problems, it feels like wasted time. Real-world parallels: database setup prechecks (skip tests if DB is down), black-box API exploration (fuzz testing), financial system integration testing. The live demo shows Claude reading docs, writing a learning test for the new Stream Send API, running it, and updating findings — all in a few minutes.

**Key Insights**:
- "The hardest part is making sure that you somehow do it earlier rather than later, but the trade-off is if I do it earlier then I'm wasting time and if I could have one-shot it I feel like I'm like, fuck, I should have just one-shot it." — Vaibhav
- "It's so fast" — Dex demos asking Claude to write a learning test; it reads docs, writes test, runs it, updates findings automatically.
- Live demo result: Claude wrote 7 tests, all passed, then updated key findings about error behavior based on actual output.
- "Now it basically becomes a really shortcut for research. Here's how I think it works. Go prove it. We won't proceed to implementation until we verify." — Dex

---

### Chunk 6: `2026-02-10-seg-08`

```yaml
chunk_id: "2026-02-10-seg-08"
episode: "2026-02-10-agentic-backpressure-deep-dive"
topic: "Developing Engineering Instinct: The Binary Search Model"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "35:10"
timestamp_end: "39:10"
tags: ["engineering-instinct", "planning-spectrum", "skill-development", "agentic-coding"]
```

**Summary**: How do you develop instinct for when to plan more vs. just vibe code? The "make the other mistake" model: instead of incrementally improving, swing to the opposite extreme. Binary search around the ideal range. This is multi-dimensional — it shifts based on the problem, the model, the day. Most people struggle with AI coding because they're "too constant with their technique" instead of adapting per-problem. Processes exist because they help 80% of people land in the good zone without needing expert instinct.

**Key Insights**:
- "Make the other mistake" — swing to the opposite extreme, then back. Binary search converges faster than incremental improvement.
- "The thing about agentic systems is you actually have to be really adaptive with the way you code. This problem, I use this technique. That problem, I use that technique." — Vaibhav
- "Most people suck at AI coding because they're too constant with their technique." — Vaibhav
- "For people managing teams: if your team isn't getting the grok of AI, that's probably because they don't have the brain cycles because they're so stressed about finishing the workload." — Vaibhav
- Dual-track exploration: "I'll have two repos open, implementing the same thing with two different strategies, one-shotting in one and planning in the other." — Vaibhav

---

### Chunk 7: `2026-02-10-seg-10`

```yaml
chunk_id: "2026-02-10-seg-10"
episode: "2026-02-10-agentic-backpressure-deep-dive"
topic: "Backpressure Defined: Automated Feedback Loops for AI"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "41:34"
timestamp_end: "46:07"
tags: ["backpressure", "feedback-loops", "compiler", "type-system", "testing", "automation"]
```

**Summary**: Backpressure = giving the model a way to fix its own mistakes via automated feedback. Layered from cheapest/fastest to most expensive: (1) type checker / compiler, (2) unit tests, (3) MCP servers / Playwright / screenshots, (4) human review. Each layer reduces what the human has to check. The best AI engineers spend days designing the backpressure harness before writing any feature code — then run Opus in a loop for days and get 20K lines of working code.

**Key Insights**:
- "Backpressure is exactly this — you give the model a way to fix its own mistakes." — Dex
- Layers: type check → compiler → unit tests → MCP/Playwright → human review. "The more layers of automated ways that the model can get feedback, the less you have to be in the loop."
- "The best AI engineers I know would spend three days designing the back pressure system, not even writing the code. They would say, here are the checks we'll run to make sure it's working. They would feed that to Opus, run it in a loop for two days, and get back 20,000 lines of working code." — Dex
- Implementation: pre-commit hooks, stop hooks (run checks when agent thinks it's done), inject failure messages back into context.
- "The backpressure mechanism doesn't have to be binary. It just needs to be observable." — Vaibhav

---

### Chunk 8: `2026-02-10-seg-11`

```yaml
chunk_id: "2026-02-10-seg-11"
episode: "2026-02-10-agentic-backpressure-deep-dive"
topic: "Visual Dependency Diagrams & CI/CD as Backpressure"
speakers: ["Vaibhav", "Dex"]
timestamp_start: "46:07"
timestamp_end: "50:26"
tags: ["dependency-graphs", "SVG", "CI-CD", "backpressure", "code-quality", "architecture-drift"]
```

**Summary**: Vaibhav demos BAML team's approach: auto-generated SVG dependency diagrams of the codebase (485 lines, diffable). Used to catch architectural drift — e.g., "bridge_cffi should not import from compiler_emits." Enforced via CI/CD: pre-commit hooks ban certain cross-package imports. Also: zero-warning builds reduce context bloat for coding agents. Two-pass accounting: review the code file AND the visual diagram to catch what you'd miss in just one.

**Key Insights**:
- SVG dependency diagram: 485 lines, passable as image to any agent, diffable as text.
- "You can pass it as an image to any agent, or because it's an SVG, it's diffable." — Vaibhav
- Warning: graph layout algorithms are unstable (adding one node can rearrange everything), so you need the image representation, not just the SVG diff.
- "Every build step that you run needs to run warning-free. Warnings cause context bloat." — Vaibhav
- "It's like two-pass accounting — you review the file, but you also make it visual, so you're checking it in two different ways." — Dex

---

### Chunk 9: `2026-02-10-seg-12`

```yaml
chunk_id: "2026-02-10-seg-12"
episode: "2026-02-10-agentic-backpressure-deep-dive"
topic: "LLM-as-Judge Critique: Why Deterministic Backpressure Wins"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "50:26"
timestamp_end: "53:12"
tags: ["LLM-as-judge", "deterministic-testing", "backpressure", "evals"]
```

**Summary**: Strong stance against over-reliance on LLM-as-judge. The builder and manager agents use the same model — you could just combine their prompts. Worse: models are steerable by framing ("is this code great?" → yes; "what's wrong?" → finds 10 issues). You cannot accidentally steer a type checker. Deterministic feedback (compiler, tests, linters) is always preferred. LLM-as-judge only makes sense if you simulate the exact conversation format from your main loop. Reviewer agents are useful for checking plan-vs-implementation deviation, but it's always a speed/accuracy tradeoff.

**Key Insights**:
- "They're both using the same model. You can accidentally steer a model. You cannot accidentally steer a type checker." — Dex
- "Ask 'is this code great?' → 'yep, it's good.' Ask 'what's wrong with this code?' → finds 10 things wrong. There's no opinions, no non-determinism in a type checker — it's either right or wrong." — Dex
- "I'm not a big fan of LLMs as judge. Don't think role prompting works for this." — Vaibhav
- "The user token does have a strong bias compared to a system token in the model." — Vaibhav (on why simulating exact conversation format matters)
- Reviewer agents are useful for plan-vs-implementation deviation checking, but "you're always making trade-offs on speed versus accuracy." — Vaibhav

---

### Chunk 10: `2026-02-10-seg-13`

```yaml
chunk_id: "2026-02-10-seg-13"
episode: "2026-02-10-agentic-backpressure-deep-dive"
topic: "Closing: Craft, Instinct, and Professional Development"
speakers: ["Dex", "Vaibhav"]
timestamp_start: "53:12"
timestamp_end: "56:01"
tags: ["takeaways", "professional-development", "craft"]
```

**Summary**: Closing reflections on developing AI engineering as a craft. Both-tracks exploration (vibe-code one path, plan the other in parallel). Uncle Bob's "professional software engineer" recipe: 45 hours/week for employer, 20 hours/week honing your craft. The instinct for when to apply which technique is the differentiator.

**Key Insights**:
- "Do it deliberately. Steer the models to the things you want. Find the things that they're really good at that's high leverage." — Dex
- "I'll have two repos open, implementing the same thing two different strategies. Through the process of doing that, I'm literally exploring both state spaces of bugs really fast." — Vaibhav
- "If you're not putting in those hours exploring what works, what's new, what's changed... the tools shift every month." — Dex
- Next episode: automating the AI That Works show's content pipeline (clip selection, highlight reel, email generation).
