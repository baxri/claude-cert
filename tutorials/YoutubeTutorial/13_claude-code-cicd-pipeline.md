# Building a Robust CI/CD Pipeline Using Claude Code: Challenges and Best Practices

Exam-critical notes from the video transcript (Domain 3 — Claude Code Configuration & Workflows, maps directly to **exam Scenario 5: Claude Code for CI/CD**).

## The Problem: CI Jobs Hanging/Timing Out

- Locally, a Claude Code session is conversational — if Claude needs clarification, it asks you and you answer
- In a CI/CD pipeline (e.g., a GitHub Action step invoking Claude to review test cases), there is **no human in the loop** — if Claude hits ambiguity and asks a follow-up question, nobody answers, the job hangs, and eventually the pipeline times out
- Root cause is not "the integration is broken" — it's that the invocation wasn't configured for **non-interactive mode**.

## The `-p` / `--print` Flag

- **The single valid mechanism for non-interactive/headless Claude Code**: `claude -p "<query>"` (short) or `claude --print "<query>"` (long form) — functionally identical
- Behavior: Claude executes on the given input without waiting for further interactive clarification
- **Exam trap:** distractor options like `--headless`, `--batch`, `--ci-mode` etc. **do not exist**. The only valid flag for this is `-p`/`--print`.

## Machine-Readable Output for Pipeline Steps

- By default Claude responds in prose — fine for humans, **useless for downstream pipeline steps** that expect structured data
- Use `--output-format json` to get structured output, and combine with `--json-schema <file>` to **strictly enforce the output shape** against a defined schema
- Practical pattern: pipe prior context (e.g., PR diff/data) into Claude via `-p`, request a specific JSON shape (e.g., `severity`, `line`, `message`, `file` fields), and write the result to a file (e.g., `findings.json`) that the next CI step consumes (e.g., via `jq`) to decide whether to block a merge

## Session Isolation — "The Psychology of Review"

- Analogy: developers and testers are separate roles not because developers can't test, but because a developer who wrote the code has an inherent bias toward believing it's correct and will tend to validate happy paths; a tester with no attachment to the implementation brings a fresh, more critical perspective
- **Same principle applies to Claude Code sessions:** if you ask Claude to write tests/review code in the **same session** where it just wrote that code, it retains the reasoning it used while writing it and tends to validate its own choices rather than critique them (may skip negative-path scenarios, etc.)
- **Fix:** use a **fresh Claude session** for the review/test step. Optionally pass a compact summary of what was done (e.g., produced via the Explore sub-agent) rather than the full prior context — the fresh session has no attachment to the original reasoning, so it acts more like an objective critic.
- **Practical CI design:** each distinct CI step (build, review, test/integration) should ideally run in its **own isolated Claude session**, not a continuation of the previous step's session.

## `CLAUDE.md` in CI/CD Pipelines

- Beyond project conventions, `CLAUDE.md` can hold **CI/CD-specific rules**: centralized code standards, review criteria, security policies, testing requirements
- Since a fresh session is loaded for each pipeline step, `CLAUDE.md` ensures that even without carrying forward a summary, each new session still gets consistent baseline context/rules automatically
- Helps balance session isolation (fresh perspective) with consistency (same standards enforced every time) without spending extra tokens re-explaining rules.

## The Deduplication Problem

- **Problem:** if each new commit triggers a fresh Claude review session with no memory of prior reviews, Claude will **re-review the entire PR from scratch** every time and likely repeat the same comments it already made — wasted tokens, redundant noise
- **Fix — pass prior findings into the fresh session:** before starting the new review session, fetch the PR's existing review comments (e.g., via the `gh` CLI) and write them to a file; pass that content into the new `claude -p` invocation as context
- With prior findings supplied, the fresh session understands what's already been reviewed and only evaluates the **new changes** (new commits/pushes) instead of redoing the whole PR
- **Exam-relevant cycle:** fetch PR → fetch previous PR comments → pass as context into a fresh session → generate only new/incremental findings → repeat on next push. This is the canonical answer to "how do you prevent Claude from redoing work / re-flagging the same issues across pipeline runs."

## Standardizing Test Generation via `CLAUDE.md`

- Without a `CLAUDE.md`, Claude generates tests using its own default judgment — inconsistent structure, mock patterns, naming across the codebase
- With testing conventions documented in `CLAUDE.md` (naming patterns, mocking rules, coverage targets), **every automated test generation run inherits these rules automatically** — consistent output regardless of which session/step generated the tests
- Document these conventions once in `CLAUDE.md` rather than re-specifying them in every prompt.

## Production Pipeline Architecture (putting it together)

A robust Claude-Code-driven CI/CD review pipeline combines all of the above:
1. **`CLAUDE.md`** — holds project/CI context, standards, testing conventions
2. **Prior findings fetch** — deduplicates by loading previous review comments before starting a new session
3. **Fresh session** — isolated from prior reasoning, informed by the prior findings for continuity
4. **`-p`/`--print`** — non-interactive execution, no hanging on unanswered questions
5. **`--output-format json` + `--json-schema`** — structured, machine-parseable output
6. **Downstream action on structured output** — e.g., parse the JSON with `jq` and block the merge if any finding has `severity: critical`

## Sample Exam Questions

- **"Which flag enables non-interactive CI mode?"** → `-p` or `--print` (reject any other fabricated flag name)
- **"Which CI use case is inappropriate for the Message Batches API?"** → blocking pre-merge checks (Batches API has up to a 24-hour window with no latency SLA — never use for blocking/latency-sensitive workflows; this ties back to the Domain 4 batch-processing rule)
- **"Why does a fresh session review better than the same session?"** → because there's no prior reasoning context biasing Claude toward validating its own earlier work

## Cheat Sheet

| Concept | Key fact |
|---|---|
| Non-interactive mode | `-p` / `--print` — the only valid flag; reject `--headless`/`--batch`/`--ci-mode` distractors |
| Structured output | `--output-format json` (+ `--json-schema` to enforce shape) — required for downstream pipeline steps to parse Claude's response |
| Session isolation | Use a fresh session for review/testing steps — same-session review inherits bias toward the code just written |
| `CLAUDE.md` in CI | Holds standards/policies/testing conventions so every fresh session inherits consistent rules |
| Deduplication | Fetch and pass prior review findings into each new session so only new changes get reviewed |
| Batches API in CI | Never appropriate for blocking pre-merge checks — no latency SLA, up to 24h processing window |
