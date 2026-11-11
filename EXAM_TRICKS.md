# Exam Tricks & Key Heuristics

Quick-reference principles for spotting the right answer fast on exam questions. Each rule maps to a recurring question pattern.

---

## Cross-Domain Mental Model (Use These First)

These are the high-level principles that decide most questions. When you see a question, check which of these applies:

| Principle | What it means |
|---|---|
| **Specificity Beats Generality** | Explicit categorical criteria > vague directives ("flag X when Y" beats "be conservative") |
| **Structure Data at Every Boundary** | Use schemas, structured errors, structured handoff packets — never free-form prose at machine boundaries |
| **Keep Contexts Isolated** | Subagents, `context: fork` skills, fresh review instances — isolation prevents pollution |
| **Programmatic Enforcement for Guarantees** | Hooks > prompts when compliance must be deterministic |
| **Fix Root Causes** | Don't paper over symptoms with prompt tweaks; identify the underlying mechanism |

---

## Domain 1 — Agentic Architecture Tricks

| Trick | When to apply |
|---|---|
| **Implement hooks for enforcement (Pre or Post tool usage)** | Compliance, policy gates, irreversible actions, audit requirements |
| **System prompts and few-shot examples operate probabilistically** | Anything requiring 100% compliance — prompts have non-zero failure rate; use hooks |
| **All interactions are managed via the coordinator** | Hub-and-spoke; subagents never talk directly to each other |
| **Subagents lack access to the shared context** | Coordinator must explicitly pass everything subagent needs in the `Task` prompt |
| **Maintain structured error context** | Subagent failures return failure_type, attempted_query, partial_results, alternatives — not generic errors |

### Quick decoder
- "Make X always happen" → **hook** (don't strengthen prompt)
- "Subagent doesn't know Y" → coordinator failed to pass Y in Task prompt
- "Workflow fails when one subagent fails" → preserve partial results, structured propagation

---

## Domain 2 — Tool Design & MCP Tricks

| Trick | When to apply |
|---|---|
| **Prioritize correcting tool descriptions initially** | First fix for any tool selection problem — not prompt instructions |
| **Begin by describing the tool** | Description is the primary selection mechanism (purpose, examples, edge cases, boundaries) |
| **Use specific tools instead of general ones** | Replace overlapping/generic tools with focused ones (`analyze_content` vs `analyze_document` ambiguity) |
| **Limit to ~five tools per agent** | Past ~18 tools selection degrades; scope per role |
| **Place `.mcp.json` in project root (or `~/.claude.json` for all projects)** | Project-level for team sharing, user-level for personal/experimental |
| **Glob searches files by path, Grep searches within file content** | Glob for filenames/extensions, Grep for code/strings inside files |

### Quick decoder
- "Tool selection unreliable" → fix descriptions (not prompts, not few-shot)
- "Agent has too many tools" → scope per subagent role
- "Generic error from tool" → use structured `isError` pattern with category + retryable + description

---

## Domain 3 — Claude Code Configuration Tricks

| Trick | When to apply |
|---|---|
| **`.claude/` for team, `~/.claude/` for personal** | Applies to commands, skills, MCP servers, CLAUDE.md |
| **Path-scoped `.claude/rules/` > subdirectory CLAUDE.md** | When conventions span multiple directories (e.g., test files everywhere) |
| **Use `context: fork` for verbose skills** | Keeps skill output from polluting main conversation |
| **Plan mode for large/multi-file/architectural work** | Direct execution only for simple, well-scoped, single-file changes |
| **Headless mode (`-p`) for CI/CD** | Otherwise CI hangs waiting for user input |
| **Independent review instance > self-review** | Same session that generated code is biased; use fresh instance |

---

## Domain 4 — Structured Output & Batch Tricks

| Trick | When to apply |
|---|---|
| **Implement structured data consistently throughout** | Schemas at every boundary — extraction, errors, handoffs |
| **Utilize `tool_use` alongside JSON Schema** | Most reliable path to schema-valid structured output |
| **Set `nullable: true` to indicate missing information** | Prevents schema-induced hallucination when source data is absent |
| **Retry operations only when facing resiliency problems** | Retry won't help if data is genuinely absent — use nullable instead |
| **It is acceptable to temporarily disable problematic categories** | High false-positive categories poison trust in all categories — better to drop than tune indefinitely |
| **For blocking workflows (pre-commit, pre-merge): synchronous API calls** | Anyone waiting → no batch |
| **Reserve Batch API for non-blocking tasks** | Overnight investigations, audits, nightly reports |

### Quick decoder
- "Want guaranteed JSON" → `tool_use` + schema + forced `tool_choice`
- "Model hallucinates missing fields" → make field nullable
- "Inconsistent output format" → add 2–4 few-shot examples
- "Pre-merge code review" → synchronous (never batch)

---

## Domain 5 — Context Management & Reliability Tricks

| Trick | When to apply |
|---|---|
| **Maintain structured error context across agents** | Subagent failures must carry failure_type, attempted_query, partial_results, alternatives |
| **Place key info at the start or end** | Lost-in-middle effect — middle of long context is unreliable |
| **Use scratchpad files for cross-session knowledge** | Survives `/compact`, restarts, lost-in-middle |
| **Subagents for verbose exploration** | Keeps file dumps and grep output out of main conversation |
| **Per-segment metrics, not aggregate** | "97% overall" hides per-vendor/per-field failures |
| **Calibrate confidence with labeled data** | Raw model confidence is unreliable; map it to actual accuracy |
| **Stratified sampling on high-confidence outputs** | Catches drift and novel error patterns in auto-accepted segments |

---

## The 5 Universal Question-Decoding Heuristics

When you read an exam question, ask:

1. **Is this asking for a guarantee?** → Programmatic (hooks), not prompts
2. **Is this asking for selection reliability?** → Tool/subagent descriptions, not prompt instructions
3. **Is this asking for output consistency?** → Few-shot examples + schema
4. **Is this asking about context loss?** → Structured persistence (case facts, scratchpads, manifests)
5. **Is this asking about coverage in research/multi-agent?** → Coverage planning before fan-out + structured propagation

---

## Top Distractor Patterns to Recognize

These are the **wrong answers that look right** — recognize them as traps:

| Distractor | Why it's wrong |
|---|---|
| "Add stronger language to the system prompt" | Probabilistic; can't deliver guarantees |
| "Lower temperature" | Doesn't fix rule-following or selection reliability |
| "Use few-shot examples to fix tool selection" | Few-shot is for output consistency, not selection — fix descriptions |
| "Tell Claude to be more conservative" | Vague directive; use explicit categorical criteria |
| "Self-review by the same session" | Carries generation bias; use independent instance |
| "Pick the most recent/active record on ambiguous match" | Privacy violation; ask for additional identifier |
| "Escalate based on detected sentiment / low confidence" | Unreliable proxies; use explicit triggers |
| "Use Batch API to save cost on pre-merge review" | Blocking workflow → no batch |
| "Give the agent all available tools" | >18 degrades selection; scope per role |
| "Resume the long session despite stale tool results" | Resume inherits stale snapshots silently; new session + summary is safer |
| "Subagents share results directly with each other" | Violates hub-and-spoke — coordinator routes |
| "Make the field required to ensure it's always extracted" | Forces hallucination when data absent — use nullable |

---

## One-Page Mental Cheat Sheet

> **For guarantees → hooks. For selection → descriptions. For consistency → few-shot + schema. For continuity → structured files. For correctness → independent review + per-segment metrics.**
>
> **Specificity > generality. Structure > prose. Isolation > sharing. Programmatic > probabilistic. Root cause > symptom patch.**
