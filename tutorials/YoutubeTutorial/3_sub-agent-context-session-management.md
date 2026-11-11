# Managing Sub-Agent Context and Session Workflow in Multi-Agent Claude Architecture

Exam-critical notes from the video transcript (Domains 1 & 5 — Agentic Architecture / Context Management & Reliability). Syllabus refs: **1.3 and 1.7** cover sub-agent invocation and session state.

## The Core Problem

- Every sub-agent starts with **zero memory** of what happened before — it does not inherit the coordinator's conversation history
- A sub-agent's entire world is exactly what you put in its `prompt` (via the task tool call) — nothing more
- This blank-slate behavior is a **feature**, not a bug — it prevents context contamination between unrelated sub-agents
- If context isn't passed properly, the sub-agent will hallucinate or produce a generic, unusable output

## Bad vs. Good Context Passing

- **Bad prompt example:** "Here are some research findings from previous agents, please synthesize them into a final report" — with no actual findings attached. The sub-agent has nothing to work from. **This is the #1 mistake in multi-agent architecture** — it looks reasonable but fails silently.
- **Good approach — structured context injection:** the coordinator builds an explicit, structured object (e.g., a JSON array of findings with source URL, title, page, retrieved-at date, confidence) and embeds it directly into the sub-agent's prompt via the task tool call, along with explicit requirements (e.g., "every claim must cite its source URL," "flag conflicting findings")

## Preserving Source Attribution (Claim-Source Mappings)

- **Problem:** summarizing multiple sources into prose loses traceability — after synthesis, zero sources remain traceable back to their origin
- **Fix — claim-source mappings:** instead of passing plain-text summaries, pass a structured object where:
  - Each **claim** has a `claim_id`, `text`, `source_id`, and `confidence`
  - Each **source** has its own `source_id` plus attribution metadata
  - Claims reference sources by `source_id`
- **Rule:** always pass the **whole object** (claims + sources) to the next agent — never extract just the claim text and drop the source. The next agent needs both to produce attributable output.

## Handling Conflicting Sources

- When two sources disagree on the same topic, **never let a sub-agent silently resolve the conflict** or silently drop one source
- **Fix:** add an explicit `conflicts` array — each conflict object has a `conflict_id`, `topic`, and the competing claims with their respective `source_id`s
- Mark resolution state as **`unresolved`** and **escalate to the coordinator** — only the coordinator makes the final call, same as a sub-agent never having authority to decide which finding "wins"
- **Direct exam pattern:** if a question asks about handling conflicting sources in a multi-agent pipeline → annotate both with attribution, mark unresolved, escalate to coordinator. Never let a sub-agent resolve it silently.

## Session Management — 3 Core Patterns

### 1. Named Session Resumption
- Start a named session; resume it later by name so Claude retains prior context
- Commands: `claude --session-name <name>` to start; `claude --resume <name>` to resume a specific session; `claude --resume` (no name) to resume the most recent session
- **Critical rule:** never resume a session without explicitly stating what has changed since last time (files modified, deps updated, requirements changed) — the agent cannot auto-detect this

### 2. Fork Session
- Analogous to Git branching — create a shared baseline session, then fork it into independent branches (Fork A, Fork B) to explore **different approaches in parallel** without disturbing the original
- Each fork inherits the same baseline context but diverges independently from that point
- **Use case:** comparing two different architectural/refactoring approaches from a shared analysis baseline

### 3. Fresh Start (with hypothesis injection)
- Start a brand-new session, but inject a **summary** of prior findings for context — don't start from a truly blank slate
- **Critical framing rule:** the injected summary must be framed as a **hypothesis to validate, not an established fact** (e.g., "codebase has changed since this analysis — treat these as hypotheses to validate, not established facts")

## Decision Rule: Resume vs. Fork vs. Fresh Start

- **Resume** when prior context is still valid AND you're continuing the exact same sequential task → use `--resume` and explicitly state specific file changes
- **Fork** when comparing new parallel approaches/architectures on a shared baseline — fork is for parallel exploration only, not for targeted updates
- **Fresh start** when tool results are fundamentally stale (e.g., large-scale codebase changes) or extended context would degrade model reasoning — inject prior findings as a hypothesis, not fact

Decision flow:
1. Are existing tool results fundamentally stale? → **Fresh session** + inject hypothesis
2. Comparing new parallel architectures? → **Fork session**
3. Continuing the exact same sequential task? → **Resume** + explicitly inform of specific file changes

## Architectural Anti-Patterns (exam-relevant)

| Anti-pattern | Correct approach |
|---|---|
| Passing prose summaries between agents | Pass structured context objects |
| Letting a sub-agent silently resolve source conflicts | Annotate both sources, mark unresolved, escalate to coordinator |
| Using fork session for a small targeted file update | Fork is for parallel exploration only — resume instead |
| Resuming a session after a heavily changed codebase | Start fresh instead |
| Resuming without informing the agent of what changed | Always explicitly declare what changed |

## Sample Scenario Breakdown

- **Multi-agent research, lost attribution:** bug = web results summarized into prose before synthesis → fix = require structured claim-to-source mappings
- **Code generation, codebase analyzed 3 days ago, one module (e.g., payments) just refactored:** fix = `--resume`, explicitly inform the agent which specific file changed to trigger targeted reanalysis
- **Code generation, codebase analyzed 1 month ago, several modules changed:** fix = start a fresh session instead of resuming

## Cheat Sheet

- Sub-agents = blank slate every time; context must be explicitly injected in the `prompt` field
- Claim-source mappings > prose summaries — preserves attribution
- Conflicts → annotate both sources, mark unresolved, escalate to coordinator (never sub-agent auto-resolves)
- Resume = same task, valid context, explicit change notes
- Fork = parallel approach comparison from a shared baseline
- Fresh start = stale context, inject prior findings as hypothesis (not fact)
