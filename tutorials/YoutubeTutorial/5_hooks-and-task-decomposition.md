# Ensuring Reliable AI Agentic Workflows with Hooks and Task Decomposition

Exam-critical notes from the video transcript (Domain 1 — Agentic Architecture & **Orchestration**, ~27-30% of the exam, wraps up Module 1). Builds directly on the capstone refund-processing project.

## The Core Problem: Prompts Are Probabilistic

- A rule in a system prompt (e.g., "never process refunds above $500") is a **request**, not a guarantee — language models are probabilistic and comply *most* of the time, not *always*
- For **critical operations** (payments, refunds, compliance), a non-zero failure rate is unacceptable — you need 100% guaranteed enforcement
- **The rule:** anything that must be guaranteed → enforce it **in code**, not just in a prompt
- **Exam trigger words:** "guarantee," "must never," "policy requires" → the answer is **programmatic enforcement / hooks**, not a stronger prompt

## What Hooks Are

- Hooks are **pieces of code that intercept the agentic loop at specific moments** — deterministic, and the model cannot override them
- Two types, tied to tool execution:
  - **Pre-tool use hook** — intercepts **before** a tool call executes. Can check prerequisites, enforce policy, or **block the tool call entirely**
  - **Post-tool use hook** — intercepts **after** a tool call executes. Typically used for **data normalization** (e.g., converting raw DB status codes to readable text, converting epoch timestamps to dates) before the result goes back to the LLM/user

**Exam pattern:**
- Policy violation / must-block scenario → **pre-tool hook**
- Data normalization / formatting scenario → **post-tool hook**

## Post-Tool Hook Example

- A `lookup_order` tool returns raw DB data (e.g., numeric status codes, epoch timestamps) — not fit for the end user
- A post-tool hook (e.g., `normalize_order_result`) intercepts the raw result, checks if the tool name matches what it cares about, and transforms the data before it re-enters the loop
- **Design rule:** a hook should be a **pass-through** for any tool it doesn't care about — early-return the raw result unchanged if the tool name doesn't match

## Pre-Tool Hook Example (Policy Enforcement)

- A pre-tool hook (e.g., `enforce_refund_policy`) only acts on the `process_refund` tool; all other tools pass through
- If refund amount > $500: return `allowed: false` with a reason (e.g., "refund exceeds $500 limit") and an escalation action (e.g., "escalate to senior support") — the tool call is **blocked entirely**, not left to the model's discretion
- If amount ≤ $500: allow execution to proceed

## Prerequisite Gates (a specific pre-tool hook pattern — high-yield for exam)

- A **prerequisite gate** is a pre-tool hook that tracks session state and **blocks a downstream tool until an upstream tool has successfully completed** and set a verified flag
- Example: `process_refund` (and `lookup_order`) must be blocked unless `get_customer` has already run and confirmed the customer exists
- Mechanism: after `get_customer` succeeds, a post-tool hook sets a session flag (e.g., `session.customer_verified = true`); the pre-tool hook checks that flag before allowing `process_refund`/`lookup_order` to execute
- **Why this matters:** a prompt instruction like "verify the customer before refunding" has a non-zero failure rate — a prerequisite gate makes the ordering a hard, code-enforced constraint
- **Classic exam scenario:** "Support agent occasionally calls `process_refund` before `get_customer` despite system prompt instructions — what most reliably guarantees proper execution ordering?" → **a prerequisite gate in code**, not a stronger prompt

## Structured Handoff Summaries

- When escalating to a human agent (or the next sub-agent), a bare message like "Customer escalated, please assist" is useless — it forces the receiving party to reconstruct context from the full conversation transcript, defeating the purpose of the agentic system
- **Fix:** before escalating, the agent should generate a **self-contained structured handoff summary** — not just raw conversation history — including things like: what happened, why it's escalating, priority, and routing tier (e.g., "senior support")
- This is a direct exam pattern: escalation without a structured handoff summary is a failure mode to recognize and correct

## Task Decomposition: Two Strategies

Before starting a multi-step agentic workflow, ask: **do you know the exact steps in advance?**

### 1. Prompt Chaining
- Break a task into a **predefined, fixed sequence** — output of step N feeds directly into step N+1 (like an assembly line / CI/CD pipeline)
- Best for: **known, predictable, compliance-sensitive tasks** — e.g., per-file code review → cross-file integration review → final report generation
- **Control level: high**
- **Warning (exam-relevant):** do NOT use prompt chaining for **open-ended tasks** — predefined steps break down the moment actual findings diverge from initial assumptions

### 2. Dynamic Decomposition
- Steps are **generated dynamically based on findings** as the agent goes — the agent decides its next investigation step based on what it discovers
- Best for: **open-ended, exploratory tasks** — e.g., investigating a production incident where you don't know the root cause up front (start with logs → decide next step based on findings → continue until root cause identified)
- **Control level: low**

**Example use cases:**
- Prompt chaining → CI/CD review pipeline, structured refund processing (customer verification → refund processing, sub-agent by sub-agent)
- Dynamic decomposition → production incident investigation, monolith-to-microservices migration

## Distractor Patterns to Reject (exam-relevant)

When a question demands a **guaranteed** outcome, these are always wrong, even if they sound reasonable:
- **"Add a stronger/clearer prompt instruction"** — prompts are never guaranteed, no matter how well-written
- **"Add few-shot examples"** — few-shot improves *consistency*, not *compliance*; it cannot replace hard enforcement for critical operations
- **"Predefine all the steps"** — doesn't work for genuinely open-ended tasks; forces dynamic decomposition instead

## Cheat Sheet

| Concept | Summary |
|---|---|
| Prompt instructions | Probabilistic, non-zero failure rate — insufficient alone for critical/financial operations |
| Pre-tool use hook | Runs before a tool executes; enforces policy, can block the call entirely |
| Post-tool use hook | Runs after a tool executes; normalizes/reshapes raw output before it re-enters the loop |
| Prerequisite gate | Pre-tool hook + session flag pattern that hard-blocks a downstream tool until an upstream tool succeeds |
| Handoff summary | Self-contained structured context passed on escalation — not a raw transcript dump |
| Prompt chaining | Fixed, known steps; sequential; high control; good for predictable/compliance tasks |
| Dynamic decomposition | Steps emerge from findings; low control; good for open-ended investigation |
| Exam trigger words | "guarantee," "must never," "policy requires" → programmatic enforcement (hooks), not prompting |
