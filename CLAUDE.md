# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository is a study workspace for the **Claude Certified Architect – Foundations** certification exam. There is no application code, build step, or test suite here — it is a collection of markdown notes, exam heuristics, and practice questions. The role of Claude here is exam tutor: quiz the user, explain concepts, clarify tradeoffs, build sample exercises, reinforce weak areas, and turn new source material (video transcripts, docs) into structured study notes.

## Repository Structure

- `CLAUDE.md` — this file (exam syllabus, domain summaries, key principles)
- `RESOURCES.md` — curated links to official Anthropic docs/courses, organized by domain
- `EXAM_TRICKS.md` — quick-reference heuristics/decision tables for spotting the right answer fast, organized by domain and by "distractor vs correct pattern"
- `some-rules.md` — recurring exam question *patterns* (e.g., "inconsistent subjective judgment → explicit criteria + examples," "prose spec keeps getting misinterpreted → switch to I/O examples") with reject/accept tables for common distractors
- `certification-preparation-questions.md` — crowdsourced questions/topics recalled by people who've taken the exam, grouped by domain, with a tiering system (Tier 1 = full Q+options recalled, Tier 2 = question only, Tier 3 = topic only). Treat "most likely" answers as opinions, not ground truth.
- `Claude Certified Architect – Foundations Certification Exam Guide.pdf` — the official Anthropic exam guide
- `tutorials/YoutubeTutorial/` — numbered study notes (`1_...md` through `7_...md`) distilled from a YouTube course covering the syllabus lesson-by-lesson (agentic loops, coordinator patterns, sub-agent context/sessions, a capstone multi-agent refund project, hooks, tool descriptions, error handling). Files are numbered in lesson/logical order — when adding a new lesson's notes, continue the numbering sequence and use a short kebab-case slug after the number.
- `.claude/skills/devops-aws/` — an unrelated DevOps skill present in this workspace; not part of the exam study material.

## Working with this Repository

- There is nothing to build, lint, or test. "Verifying" work here means checking that notes are accurate against the source material and internally consistent with the exam guide and `EXAM_TRICKS.md`/`some-rules.md`.
- When distilling a new transcript/lesson into notes, follow the existing pattern in `tutorials/YoutubeTutorial/`: summarize into a titled markdown file with exam-relevant headers, a "cheat sheet" table, and explicit exam trigger words / distractor patterns where the source calls them out. Don't reproduce the transcript verbatim — summarize and structure it.
- Don't invent exam questions or answers; when adding to `certification-preparation-questions.md`-style content, keep the existing tiering/uncertainty conventions (mark speculative answers, cite when something is "most likely" vs confirmed).

---

## Exam Overview

- **Format:** Multiple-choice, 4 options, 1 correct answer per question
- **Pass score:** 720 / 1000 (scaled)
- **Scenarios:** 4 of 6 presented randomly per sitting

### Domain Weights

| Domain | Weight |
|---|---|
| 1 – Agentic Architecture & Orchestration | 27% |
| 2 – Tool Design & MCP Integration | 18% |
| 3 – Claude Code Configuration & Workflows | 20% |
| 4 – Prompt Engineering & Structured Output | 20% |
| 5 – Context Management & Reliability | 15% |

### The 6 Exam Scenarios

1. **Customer Support Resolution Agent** – Agent SDK + MCP tools (get_customer, lookup_order, process_refund, escalate_to_human); 80% first-contact resolution target
2. **Code Generation with Claude Code** – CLAUDE.md, slash commands, plan mode; team dev workflow
3. **Multi-Agent Research System** – Coordinator + specialized subagents; web search, doc analysis, synthesis, report generation
4. **Developer Productivity with Claude** – Built-in tools (Read/Write/Bash/Grep/Glob); MCP server integration
5. **Claude Code for CI/CD** – `-p` flag, `--output-format json`, structured PR feedback; automated code review
6. **Structured Data Extraction** – JSON schema tool_use, validation-retry loops, batch processing

---

## Domain 1: Agentic Architecture & Orchestration (27%)

**Core loop pattern:**
- Send request → check `stop_reason` → if `"tool_use"`: execute tool, append result, loop → if `"end_turn"`: done
- Anti-patterns: parsing natural language to detect termination; arbitrary iteration caps; checking for assistant text content

**Multi-agent coordinator patterns:**
- Hub-and-spoke: coordinator manages all inter-subagent communication and routing
- Subagents have **isolated context** — they do NOT inherit parent history
- Coordinator decomposes tasks, delegates, aggregates, decides which subagents to invoke
- Parallel subagent execution: emit multiple `Task` tool calls in **one coordinator response**
- `allowedTools` must include `"Task"` for a coordinator to spawn subagents

**Workflow enforcement:**
- Programmatic prerequisites > prompt instructions for deterministic compliance (e.g., block `process_refund` until `get_customer` returns verified ID)
- Hooks (`PostToolUse`, tool call interception) for guaranteed business rule enforcement
- Prompt-only instructions have non-zero failure rate — insufficient for financial operations

**Task decomposition:**
- Fixed sequential (prompt chaining) for predictable multi-aspect reviews
- Dynamic adaptive for open-ended investigation tasks
- Split large code reviews: per-file local passes + separate cross-file integration pass

**Session management:**
- `--resume <session-name>` to continue named sessions
- `fork_session` for divergent branches from shared analysis baseline
- Start fresh with structured summaries when prior tool results are stale

---

## Domain 2: Tool Design & MCP Integration (18%)

**Tool descriptions are the primary mechanism for tool selection** — minimal descriptions → unreliable selection among similar tools.

Good tool descriptions include:
- Purpose and expected inputs/outputs
- Example queries
- Edge cases and boundary explanations
- When to use it vs similar alternatives

**Structured error responses (MCP `isError` pattern):**
```json
{
  "isError": true,
  "errorCategory": "transient|validation|permission|business",
  "isRetryable": false,
  "description": "human-readable explanation"
}
```
- Distinguish: access failures (timeout → retry decision) vs valid empty results (success, no matches)
- Subagents handle transient failures locally; propagate only what they can't resolve + partial results

**Tool distribution:**
- Too many tools (e.g., 18) degrades selection reliability — give agents only role-relevant tools
- `tool_choice: "auto"` — model may return text; `"any"` — must call a tool; forced: `{"type": "tool", "name": "..."}` — must call specific tool

**MCP configuration:**
- Project-scoped: `.mcp.json` (committed, shared with team)
- User-scoped: `~/.claude.json` (personal/experimental)
- Credentials: use `${ENV_VAR}` expansion in `.mcp.json`, never hardcode
- MCP resources expose content catalogs to reduce exploratory tool calls

---

## Domain 3: Claude Code Configuration & Workflows (20%)

**CLAUDE.md hierarchy (most-specific wins):**
1. User-level: `~/.claude/CLAUDE.md` — NOT shared via version control
2. Project-level: `.claude/CLAUDE.md` or root `CLAUDE.md`
3. Directory-level: subdirectory `CLAUDE.md` files

**Modular organization:**
- `@import` syntax to reference external files
- `.claude/rules/` for topic-specific rule files (e.g., `testing.md`, `api-conventions.md`)
- Path-scoped rules in `.claude/rules/` with YAML frontmatter:
  ```yaml
  paths: ["**/*.test.tsx"]
  ```
  Loads only when editing matching files — better than subdirectory CLAUDE.md for conventions spanning multiple directories

**Custom slash commands and skills:**
- Project-scoped: `.claude/commands/` (version-controlled, team-wide)
- User-scoped: `~/.claude/commands/` (personal)
- Skills in `.claude/skills/` with `SKILL.md` frontmatter options:
  - `context: fork` — runs in isolated sub-agent, prevents polluting main conversation
  - `allowed-tools` — restricts tool access during skill execution
  - `argument-hint` — prompts for required parameters when invoked without args

**Plan mode vs direct execution:**
- **Plan mode:** large-scale changes, multiple valid approaches, architectural decisions, multi-file modifications (e.g., monolith-to-microservices, 45+ file migrations)
- **Direct execution:** simple, well-scoped, single-file changes (e.g., adding a date validation check)

**CI/CD integration:**
- `-p` / `--print` flag: non-interactive mode (required in CI to prevent hangs)
- `--output-format json --json-schema` for machine-parseable structured findings
- Use independent Claude instance for reviewing code (same session = less effective self-review)

---

## Domain 4: Prompt Engineering & Structured Output (20%)

**Precision over vagueness:** "flag comments only when claimed behavior contradicts actual code behavior" > "check that comments are accurate"

**Few-shot examples:**
- Most effective technique when detailed instructions alone produce inconsistent results
- Create 2-4 targeted examples for ambiguous scenarios showing reasoning
- Include examples that demonstrate specific output format (location, issue, severity, fix)
- Use to demonstrate correct handling of varied document structures

**Structured output via `tool_use`:**
- `tool_use` with JSON schema = most reliable approach, eliminates syntax errors
- Does NOT prevent semantic errors (e.g., line items not summing to total)
- Schema design: nullable fields prevent hallucination when information may be absent; enum with `"other"` + detail string for extensible categories

**Validation-retry loops:**
- On failure: send original doc + failed extraction + specific validation errors
- Retries work for format/structure errors; ineffective when information is simply absent

**Batch processing (Message Batches API):**
- 50% cost savings; up to 24-hour processing window; no latency SLA
- Appropriate for: overnight reports, weekly audits, nightly test generation
- NOT appropriate for: blocking pre-merge checks
- Does NOT support multi-turn tool calling within a single request
- Use `custom_id` for request/response correlation

**Multi-pass review:**
- Independent review instance > self-review (model retains reasoning context from generation)
- Per-file local analysis + separate cross-file integration pass to prevent attention dilution

---

## Domain 5: Context Management & Reliability (15%)

**Context preservation:**
- Extract transactional facts (amounts, dates, order numbers) into persistent "case facts" block
- Trim verbose tool outputs to only relevant fields before accumulation
- "Lost in the middle" effect: place key findings at the beginning; use section headers
- Use scratchpad files to persist findings across context boundaries

**Escalation triggers:**
- Customer explicitly requests a human → escalate immediately (no investigation first)
- Policy is ambiguous or silent on the customer's specific request
- Unable to make meaningful progress
- Multiple customer matches → ask for additional identifiers (not heuristic selection)
- Sentiment-based escalation and self-reported confidence scores are **unreliable proxies**

**Error propagation in multi-agent systems:**
- Return structured context: failure type, attempted query, partial results, alternative approaches
- Never silently suppress errors (returning empty as success) or terminate entire workflows on single failures
- Subagents implement local recovery for transient failures; propagate only what they can't resolve

**Large codebase exploration:**
- Spawn subagents for verbose discovery phases; main agent coordinates at high level
- `/compact` to reduce context during extended sessions
- Structured state exports (manifests) for crash recovery: each agent exports state, coordinator loads on resume

**Human review workflows:**
- Field-level confidence scores + calibration using labeled validation sets
- Stratified random sampling of high-confidence extractions for ongoing error rate measurement
- Analyze accuracy by document type AND field before reducing human review

**Information provenance:**
- Require subagents to output structured claim-source mappings (URL, excerpt, publication date)
- Conflicting statistics: annotate with both values + source attribution (don't arbitrarily pick one)
- Distinguish well-established findings from contested ones in synthesis output

---

## Key Principles to Internalize (Cross-Domain)

1. **Programmatic enforcement > prompt instructions** for deterministic compliance requirements
2. **Tool descriptions > few-shot examples** as the first fix for tool selection reliability
3. **Explicit criteria > vague instructions** (e.g., "flag X only when Y" > "be conservative")
4. **Independent review instance > self-review** for code quality checks
5. **Batch API only for latency-tolerant workloads** — never for blocking workflows
6. **nullable fields prevent hallucination** — required fields force fabrication when data is absent
7. **Parallel subagent calls = one coordinator response** with multiple Task tool calls
8. **`context: fork` isolates skills** — prevents verbose output from polluting main session
9. **`.claude/rules/` with glob patterns > subdirectory CLAUDE.md** for cross-directory conventions
10. **Structured error context > generic errors** — enables intelligent coordinator recovery

---

## Out-of-Scope (Will NOT Appear)

Fine-tuning, model training, authentication/billing, infrastructure/cloud deployment, Constitutional AI, embeddings/vector DBs, computer use, vision, streaming API, OAuth, rate limiting, token counting, prompt caching implementation details.

---

## How to Use This Workspace

Ask Claude to:
- **Quiz you** on a specific domain or scenario ("Quiz me on Domain 1")
- **Explain a concept** ("Explain when to use `context: fork`")
- **Analyze a sample question** ("Why is B wrong in Q2?")
- **Build a practice exercise** ("Build Exercise 1 from the exam guide")
- **Identify weak areas** ("I keep confusing tool_choice options — drill me")
- **Turn a lesson transcript into notes** ("Next topic is [transcript]") — produces a new numbered file in `tutorials/YoutubeTutorial/`
