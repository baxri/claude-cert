# Agentic Loops & stop_reason Explained

Exam-critical notes from the video transcript on agentic loops (Domain 1 — Agentic Architecture & Orchestration).

## Roles

- **user** — messages from the end user
- **assistant** — Claude's responses
- **system** — Claude's persona/instructions, set once at the start of a conversation
- The API requires roles to be **explicitly set** on every message (unlike chat UIs, where this happens invisibly)

## Chat vs. Agent

- **Chat:** message → response → done (one-shot)
- **Agent:** message → reason → act (tool call) → observe result → reason again → repeat until finished (the **agentic loop**)

## Core Loop Mechanics

- The Claude API is **100% stateless** — every call must include the **entire conversation history**, or context is lost
- Each response includes a `stop_reason`. Only two values matter for loop control:
  - **`tool_use`** → Claude wants to call a tool. Execute it yourself, append the result, continue the loop.
  - **`end_turn`** → Claude is finished. Extract final text and exit. **This is the only valid loop exit condition.**
- Message ordering: append **assistant message first** (contains the `tool_use` block), then **user message** (contains the `tool_result`) — tool results are sent back to Claude as if from the user
- The `tool_use_id` in the result must exactly match the id from the tool_use block

## Anti-Patterns (high-yield for exam)

1. **Parsing natural language** to detect completion (e.g., checking if text contains "I'm done") — wording is inconsistent and unreliable
2. **Using iteration caps as the primary stop condition** — a max-iteration limit (e.g., 50) should only be a **safety cap**, never the mechanism that decides the loop is done
3. **Content-type checking** (text vs. JSON vs. boolean) as a stand-in for `stop_reason` — they measure entirely different things

## Model-Driven Tool Selection

- Claude decides **when** and **which** tool to call — your code never hardcodes tool-triggering logic
- Your code only **executes** the requested tool and returns the result; Claude cannot call functions directly
- Tool **descriptions** are what drive correct tool selection by the model

## Cheat Sheet

| Concept | Rule |
|---|---|
| Loop exit | `end_turn` is the **only** valid exit |
| Tool call | `tool_use` → execute → append result → continue |
| Append order | assistant (tool_use) → user (tool_result) |
| IDs | `tool_use_id` must match exactly |
| API state | Stateless — always send full message history |
| Anti-patterns | No text parsing, no iteration-cap-as-primary-exit, no content-type checks |
