# Capstone: Implementing an Agentic Multi-Agent Framework for Customer Refund Processing

Exam-critical notes from the video transcript — a hands-on capstone combining Episodes 1–3 (agentic loop, coordinator/multi-agent patterns, context & session management) into one working project. Maps directly to **Exam Scenario 1: Customer Support Resolution Agent**.

## Scenario

Customer `C001` requests a refund for order `0100`. A **coordinator** orchestrates two sequential sub-agents:

1. **Customer verification sub-agent** — verifies the customer exists, passes a structured finding forward
2. **Refund processor sub-agent** — receives the verification finding explicitly, looks up the order, confirms it belongs to the customer, and processes the refund if eligible

The coordinator aggregates results and prints the final summary.

## How the Pieces Map to Prior Lessons

- **Agentic loop (Ep. 1):** each sub-agent runs its own independent agentic loop — checks `stop_reason`, executes tools on `tool_use`, appends assistant message then tool result, exits only on `end_turn`
- **Coordinator/delegation (Ep. 2):** coordinator decomposes the refund request into two subtasks and delegates each to a scoped sub-agent
- **Structured context passing (Ep. 3):** the verification sub-agent's result is passed to the refund sub-agent as an explicit typed object (not a vague prose summary) — "context must be passed," not assumed

## Tool Registry Used

- `get_customer` — verifies a customer by `customer_id` (required)
- `lookup_order` — looks up an order, confirms it belongs to the customer
- `process_refund` — issues a refund for an eligible order

Each sub-agent only gets the tools relevant to its scoped task (verification sub-agent gets `get_customer`; refund sub-agent gets `lookup_order` + `process_refund`) — reinforcing **tool distribution**: give agents only role-relevant tools.

## Why Structured Handoff Matters (demonstrated live)

- **Happy path (`C001`, order `0100`):** verification sub-agent confirms customer "Alice" exists → structured finding passed to refund sub-agent → order looked up, confirmed delivered/eligible → refund processed successfully → coordinator aggregates and reports success
- **Failure path (`C99`, nonexistent customer):** verification sub-agent returns "customer does not exist" → the coordinator **does not invoke the refund sub-agent at all** — the pipeline stops at the failed verification step
- **Key lesson:** without structured inputs/context and without gating on the verification result, the second sub-agent could still run and produce a **hallucinated** result (e.g., fabricating a refund for a customer that doesn't exist). Passing structured, explicit findings — and gating downstream execution on them — prevents this.
- This is a concrete illustration of the CLAUDE.md principle: **programmatic prerequisites > prompt instructions** for deterministic compliance (e.g., blocking `process_refund` until `get_customer` returns a verified ID) — critical for financial operations.

## Important Implementation Caveat (exam-relevant)

- This capstone code does **not** use an actual `task` tool call to spawn sub-agents — it uses plain Python functions to run each sub-agent's loop
- **Why:** the `task` tool for spawning sub-agents is a **Claude Code / Agent SDK feature**, not something available in the raw Anthropic Python SDK
- The tool definitions/schemas and architectural patterns (decomposition, structured handoff, aggregation) are still correct and exam-relevant — but be aware that "spawning via task tool calls" is specifically an Agent SDK/Claude Code concept, not a raw-SDK one. **For the exam, the concepts and patterns matter, not the literal implementation mechanism.**

## Cheat Sheet

- Coordinator decomposes → delegates → passes structured findings between sub-agents → aggregates → reports
- Each sub-agent = its own independent agentic loop (same `stop_reason` rules as any agent)
- Gate downstream sub-agents on upstream verified results — don't let a later sub-agent run on missing/failed context
- Only raw Anthropic SDK usage here — real `task` tool spawning is a Claude Code/Agent SDK capability
- Unverified/ungated handoffs risk hallucinated downstream results — a direct real-world case for programmatic enforcement over prompt-only trust
