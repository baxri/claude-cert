# Enhancing AI Agent Tool Usage with Effective Descriptions and System Prompts

Exam-critical notes from the video transcript (Domain 1/2 — Agentic Architecture & Tool Design). Directly powers **official sample question 2** from the exam syllabus.

## Why Tools Get Misused in Production

Symptoms after shipping an otherwise well-built agent (proper tools, hooks, coordinator, sub-agents):
- Agent looks up an order but never processes an eligible refund
- Agent escalates to a human even for cases well within its authorization (e.g., refunds under the $500 limit)

**Root cause in most cases:** the **tool description** quality — not a missing hook. Pre-tool hooks control *whether an already-selected tool is allowed to run* (policy enforcement); they don't help if Claude never selects the right tool in the first place.

## How Claude Picks a Tool

- There is **no separate routing layer or algorithm** — Claude picks a tool purely by reading its **description** at that point in the agentic loop
- Tool descriptions are **not developer documentation** — they exist for Claude's runtime judgment, not for humans reading the code later
- Analogy: a restaurant menu with items literally named "Dish A / Dish B" is useless; well-named, well-described items ("Paneer Tikka Masala") let the customer choose correctly. Same logic applies to tool descriptions and Claude.

## Anatomy of a Production-Grade Tool Description

A good description should specify:
1. **What the tool does**
2. **When to call it** (including ordering relative to other tools)
3. **What NOT to use it for** (negative constraints)
4. **What it returns**

Example pattern (`get_customer`):
> "Retrieve a verified customer profile by customer ID. Returns name, email, plan tier, and account standing. Always call this first before any other tool — required for customer verification before order lookups or refunds."

Example pattern (`process_refund`):
> "...Requires: call `get_customer` first to verify identity, call `lookup_order` to confirm the order exists and belongs to the customer. Only call this after both checks pass."

**Takeaway:** a one-line description is not production-grade. This exact pattern — writing rich, ordered, constrained tool descriptions — is directly tested (sample question 2 in the official syllabus).

## Description Overlap (a major cause of tool misrouting)

- Common in multi-agent research scenarios (**exam Scenario 3**: coordinator with web search, document analysis, and synthesis tools)
- **Problem example:** three tools all described using near-synonyms ("search for information," "search documents for information," "analyze and extract information from content") — to Claude these read as interchangeable, causing misrouting
- **Fix:** make descriptions **mutually exclusive** — each tool's description should stake out a distinct, unambiguous domain:
  - Web search → "query live pages via search engine, for current events / URLs not yet in the corpus; do not use for documents already in the research corpus"
  - Document search → "full search across the preloaded research corpus"
  - Analyze content → "deep analysis of specific content already retrieved" (post-retrieval processing)
- **Exam pattern:** when a question describes tool misrouting/confusion, the distractor options often suggest "add few-shot examples" — **the correct fix is nearly always to rewrite/disambiguate the tool descriptions first**, not add examples.

## Tool Naming (overlap can be baked into the name itself)

- Generic tool names (`search`, `get_data`, `fetch_info`, `process`, `handle_request`) carry no semantic weight and increase misrouting risk, independent of the description
- Fix: give tools specific, purpose-named identities — e.g., `search_live_web`, `lookup_customer_profile`, `retrieve_order_history`, `get_customer_by_id`, `list_customer_orders` — analogous to naming DAO/repository methods precisely rather than generically
- **Diagnostic order when a tool isn't firing correctly / misroutes:** check naming and description first. Only route to hooks for policy violations, and to post-tool hooks for formatting — those solve different problems.

## System Prompt Alignment

- A generic system prompt (e.g., "You are a customer support assistant") biases Claude toward certain tool categories by default — e.g., it may lean toward `get_customer` lookups and under-use `process_refund` even when a refund is clearly appropriate
- **Role framing in the system prompt creates a bias toward certain tool categories** — this bias must be counteracted by aligning the system prompt with the full scope of the agent's job, not just a narrow label
- **Fix:** enrich the system prompt with:
  - A clearer **role/goal statement** (e.g., "your goal is contact resolution of returns and refunds," not just "customer support assistant")
  - An explicit **tool-use sequence**, e.g.: "Always call `get_customer` first → then `lookup_order` → then `process_refund` if the order is eligible → escalate to human only as a last resort"
- System prompt and tool descriptions must be **mutually aligned** — a narrow system prompt undercuts even well-written tool descriptions.

## Cheat Sheet

| Symptom | Likely fix |
|---|---|
| Wrong tool selected / tool never called | Rewrite tool description (what/when/when-not/returns) |
| Tools confused with each other | Make descriptions mutually exclusive, disambiguate domains |
| Generic tool names causing confusion | Rename tools to be specific and purpose-scoped |
| Agent biased toward wrong tool category by default | Enrich system prompt with goal + explicit tool-use sequence |
| Tool allowed to run but shouldn't (policy) | Pre-tool hook (not a description fix) |
| Raw tool output needs formatting | Post-tool hook (not a description fix) |
| Exam distractor to reject | "Add few-shot examples" when the real issue is tool description/naming quality |
