# Multi-Agent Systems & Coordinator Patterns Explained

Exam-critical notes from the video transcript (Domain 1 — Agentic Architecture & Orchestration).

## Why Single Agents Fail

- **Context overflow** — one agent accumulates all tool results/history, hitting the context window before the task finishes
- **Sequential bottlenecks** — a single agent can only use tools one at a time, so independent subtasks queue up and latency compounds
- **Specialization gap** — a jack-of-all-trades agent (research + write + everything) produces generic results instead of expert-level output

**Fix:** multi-agent architecture — specialized agents running in parallel, each with a scoped context.

## Hub-and-Spoke Topology

- One **coordinator** (central node) + multiple independent **sub-agents**
- **Routing mandate:** all communication routes exclusively through the coordinator — sub-agents never talk to each other and don't know other sub-agents exist
- Sub-agents do their scoped work and report results back to the coordinator only
- The coordinator — not the sub-agents — produces the final synthesized output

## Coordinator Responsibilities (4)

1. **Task decomposition** — break the complex request into scoped, assignable pieces
2. **Delegation** — spawn the right sub-agent for each piece
3. **Result aggregation** — collect independent outputs and synthesize into one unified result
4. **Error handling** — gracefully handle sub-agent failures (e.g., a sub-agent's own stop_reason/failure signal)

## The Task Tool & allowedTools

- The **`task`** tool is how a coordinator spawns sub-agents
- A coordinator's `allowedTools` list **must explicitly include `"Task"`** (capital T) — without it, the coordinator cannot delegate/spawn anything
- Standard sub-agents (workers) must **not** have task tool access — otherwise every agent could spawn agents, breaking the hub-and-spoke model
- **Hierarchical sub-coordinators** are allowed *only* if a node is explicitly designed to delegate to a deeper tertiary layer

## Agent Definition (Task Tool Payload) — 4 fields

When the coordinator calls the task tool, it must pass a strict **agent definition** object:

1. **description** — the sub-agent's exact purpose/role (a label)
2. **prompt** — the specific instructions, goal, and criteria for that sub-agent
3. **allowedTools** — the explicit, isolated list of tools that specific worker may access
4. **model** — which model the sub-agent uses

Without this object, the sub-agent cannot be successfully spawned.

## Parallel vs. Sequential Sub-Agent Spawning

- **Exam fact:** to spawn sub-agents in parallel, the coordinator emits **multiple task tool calls in a single coordinator response** — not across separate turns
- **Parallel:** total time ≈ max(S1, S2, S3, ...) — bounded by the slowest sub-agent
- **Sequential (anti-pattern):** total time ≈ S1 + S2 + S3 + ... — additive, defeats the purpose of going multi-agent
- **Exam diagnostic alert:** if a multi-agent system runs ~3x slower than expected, suspect the coordinator is committing the **sequential anti-pattern** (spawning one sub-agent per response instead of batching multiple task calls into one response)

## Context Isolation ("the gotcha")

- **Sub-agents always start with a blank context** — they do NOT inherit the coordinator's conversation history
- If a sub-agent needs background info, the **coordinator must explicitly inject it** into the sub-agent's `prompt` field in the agent definition
- **Exam diagnostic:** if a sub-agent produces irrelevant, duplicated, or context-unaware output, the root cause is almost always **missing explicit context injection** by the coordinator — not a model failure

## Good vs. Bad Sub-Agent Prompting

- **Good:** specify the **goal/outcome** (e.g., "find studies contradicting X finding") — let the sub-agent's LLM reason about how
- **Bad (anti-pattern):** dictate rigid **step-by-step instructions** (e.g., "go to Google Scholar, then do X, then Y") — brittle; fails if the specific path breaks (site down, URL changed) and prevents the LLM from adapting intelligently
- Guardrails/constraints belong in the prompt, but as goals/criteria — not as a rigid script

## Architecture Cheat Sheet

| Concept | Rule |
|---|---|
| Communication | All routes through the coordinator — enables full observability |
| Task tool | Coordinator needs it; standard sub-agents must not have it |
| Parallel spawning | Multiple task tool calls in **one** coordinator response |
| Sequential = anti-pattern | One task call per response → ~3x+ slower |
| Context isolation | Sub-agents start blank; coordinator must inject context explicitly in the prompt field |
| Prompting sub-agents | Specify goals/criteria, not step-by-step scripts |
| Aggregation | Coordinator (or a dedicated synthesis sub-agent) must merge sub-agent outputs into one coherent result |
