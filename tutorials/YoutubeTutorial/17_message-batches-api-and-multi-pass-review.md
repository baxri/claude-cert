# Cost-Effective Code Review and Test Generation Using Anthropic's Message Batches API

Exam-critical notes from the video transcript (Domain 4 — Prompt Engineering & Structured Output, part of the 20%-weighted domain). Covers two related topics: the Message Batches API, and multi-pass/independent-instance review architecture.

## The Scenario: Paying Full Price for Work Nobody's Waiting On

- A CI/CD code review pipeline and a nightly test-generation pipeline both call Claude at full standard-API price, even though **nobody is waiting on the results in real time** (PRs opened late in the day, nightly test runs checked the next morning)
- Anthropic offers the same model/output quality at **50% lower cost** if you're willing to accept **asynchronous, delayed delivery** — this is the **Message Batches API**.

## Message Batches API — Core Mechanics

- Analogy: standard Messages API = ordering at a restaurant (immediate, full price); Batches API = placing a large catering order in advance (delayed, discounted)
- You submit a large list of requests (each tagged with a **custom ID** — see below) for asynchronous, non-blocking processing
- **Delivery window: up to 24 hours.** This is a maximum, not a guarantee of a specific time — a batch could complete in minutes or take nearly the full 24 hours. There is **no SLA** on exact timing.
- **50% cost discount** compared to standard Messages API pricing, in exchange for accepting unpredictable timing

### Batches API Limitations (exam-relevant)

- **Single-turn only** — no multi-turn conversation or iterative back-and-forth within a batch request; you submit once and get one response back, no follow-up correction loop
- **No streaming support** (doesn't apply to an async, delayed-delivery model)
- **No delivery-time guarantee** — cannot be relied upon for anything with a real-time or bounded-time dependency

## Decision Rule: Batch vs. Standard API

Ask, in order:
1. **Is anyone (a person or a pipeline) actively waiting for the result right now?** → If yes, use **standard API**.
2. **Does the task require multiple back-and-forth turns / multi-turn tool calling** (e.g., an agentic loop where a sub-agent calls a tool, gets feedback, calls another tool)? → If yes, use **standard API**.
3. If neither applies → **Batches API** is appropriate.

**Good fits:** overnight code review reports, nightly test generation, security audits, batch-processing a large set of independent items (e.g., 500 invoices) over a weekend.

**Bad fits:** anything blocking a CI/CD step where the next step needs the output to proceed (a GitHub Action can't wait up to 24 hours), any agentic workflow needing iterative tool-call feedback.

## Custom ID — Correlating Requests and Responses

- Every request submitted in a batch **must have a unique custom ID** — this is how you match a given response back to the request that produced it once the batch completes
- The batch's output carries the **same custom ID** back with each result
- **Exam tip:** whenever a question asks how to correlate batch request/response pairs, the answer is always **custom ID**.
- **Good custom ID practice:** meaningful and unique — e.g., a PR number, a file path + version, not a random UUID (which conveys no context and makes debugging/traceability harder)
- Conceptually the same idea as correlation IDs in other async systems (e.g., AWS SQS message IDs, polling an async agent job by its task ID) — any asynchronous operation needs a way to reconnect a result to its originating request.

## Sample Exam Pattern: Batch API Misuse

**Q:** A team's Claude-powered CI/CD code review currently blocks PR merges. To cut costs, a developer suggests switching to the Batches API. What's the primary concern?
**A:** The Batches API has **no delivery guarantee/SLA** — it could take up to 24 hours, which is incompatible with a blocking merge-gate workflow. (This compounds with the earlier point: if the pipeline also needs multi-turn tool interaction across steps, that's a second, independent reason batch doesn't fit.)

**Q (fit case):** A pipeline extracts sentiment/topics from 2,000 customer feedback forms weekly; each form is processed independently, no tools are needed, and it can run as a nightly batch.
**A:** Excellent fit for the Batches API — independent, non-blocking, single-turn, no tools, and the 50% savings apply directly. (Watch for a "real time" mention elsewhere in the question that's actually describing when the *source data* arrives, not a hard requirement that *processing* happen in real time — read carefully for what's actually blocking vs. what merely occurs on a schedule.)

---

## The Self-Review Trap

- If the same Claude session/instance that **wrote** a piece of code is also asked to **review** or write tests for it, that instance retains its original conversation history and reasoning context — it's biased toward validating decisions it already made, the same way a developer reviewing their own code tends to assume it's correct rather than actively hunting for flaws
- **Fix:** use an **independent session/instance** for the review step — a fresh session with no memory of the original implementation reasoning approaches the code like a QA engineer would: no attachment to prior decisions, evaluating it fresh
- **Cost consideration:** starting a completely blank new session for review can be expensive if it has to re-derive everything from scratch. Mitigate this by feeding the fresh review session a **compact summary** (e.g., produced via an Explore sub-agent or a compaction step) rather than the full turn-by-turn history — giving it functional context (what changed, what it should verify) without giving it the original reasoning that would bias it.
- **Exam signal:** when a question asks how to improve review/code quality, the answer involves **splitting into independent review instances**, not prompting the same instance to "double check" or "review more carefully" — self-instruction to be more careful doesn't remove the underlying bias.

## Multi-Pass Review Architecture — Depth + Breadth

1. **Per-file review pass** — independent Claude instances/sessions each reviewing one file in parallel, focused on style/bugs/security within that file (**depth**)
2. **Cross-file integration pass** — a separate instance collates the findings from all per-file reviews and checks how the files interact as a whole (**breadth**) — analogous to how unit tests validate individual components while integration tests validate that components correctly work together (e.g., an auth service correctly gating a cart service)
3. Each instance starts with a **clean context**, eliminating self-review bias at every stage

**Takeaway:** per-file reviews give depth; cross-file review gives breadth; independent instances at each stage eliminate self-review bias — this combination is the answer whenever an exam question describes shallow or cross-file-blind review feedback from a single Claude instance.

## Enterprise CI/CD Synthesis (putting batches + multi-pass together)

- **Real-time/blocking pipeline (e.g., PR review gating a merge):** must use the **standard Messages API** (not batch, because it's blocking) — run a multi-pass review (parallel per-file instances → cross-file integration instance), and include prior review findings in the prompt to avoid re-flagging already-reviewed issues (deduplication pattern from the earlier CI/CD lesson) — results in minutes, at standard (higher) cost.
- **Nightly/non-blocking pipeline (e.g., midnight cron job for test generation, full codebase security audit, documentation generation):** ideal for the **Batches API** — non-blocking, within the 24-hour SLA window, up to 50% cost savings. Every review instance in this pipeline is still independent (no self-review), same principle as the real-time pipeline, just running asynchronously.

## Cheat Sheet

| Fact | Detail |
|---|---|
| Batches API timing | Up to 24 hours, no exact-time SLA |
| Batches API turn support | Single-turn only — no multi-turn tool calling |
| Batches API cost | ~50% discount vs. standard Messages API |
| Never use batch when | A pipeline or person is actively blocked/waiting for the result |
| Correlating batch requests/responses | Custom ID — unique and meaningful (e.g., PR number, file path+version), not a random UUID |
| Self-review | Always biased — retains the original reasoning/context |
| Fix for review bias | Independent session/instance — "fresh eyes," optionally seeded with a compact summary to control cost |
| Best review architecture | Multi-pass: per-file (depth) + cross-file integration (breadth), independent instances throughout |
