# Effective Error Handling and Tool Management in Agentic AI

Exam-critical notes from the video transcript (Domain 2 — Tool Design & MCP Integration, syllabus tasks ~2.2/2.3). Builds on the capstone refund-processing agent.

## The Silent Failure Problem

- Unlike traditional code with try/catch blocks, an LLM-driven agent has no built-in concept of "error" unless you explicitly tell it one occurred
- If a tool failure isn't clearly signaled, the model may treat the failed response as a **success** and hallucinate a plausible-sounding result, or worse, the whole pipeline can break
- **The fix — the MCP `isError` flag pattern:** every tool result should explicitly set `isError: true` when something goes wrong, so the model knows to reason about recovery rather than treating the response as valid data
- **Anti-pattern:** returning an empty array (or any "quiet" fallback) on failure — the agent will treat it as a valid, if unhelpful, success

## Four Error Categories (high-yield for exam)

The exam tests whether each category is **retryable**:

| Category | Meaning | Retryable? |
|---|---|---|
| **transient** | Timeouts, network blips, temporary service/DB unavailability | **Yes** — likely to succeed on retry |
| **validation** | Invalid/malformed input parameters | Sometimes — retrying with the *same* input won't help; retrying with *corrected* input can. The underlying issue must be fixed, not blindly retried |
| **business_logic / policy** | Action blocked by a business rule (e.g., refund > $500 policy) | **No** — retrying doesn't change the policy outcome |
| **permission** | Agent lacks authorization/access to the tool or resource | **No** — retry won't grant access |

**Exam trap:** for validation errors, "retry is possible" is technically true but incomplete — the correct framing is that the underlying issue must be fixed first, not that blind retries will eventually succeed.

## Structured Error Metadata

A bare `isError: true` with a generic message ("operation failed") gives the coordinator nothing actionable — it can't judge whether to retry, escalate, or reroute.

A **good structured error response** includes:
- `errorCategory` (transient / validation / business_logic / permission)
- `isRetryable` (boolean)
- A human-readable `description` for debugging/coordinator reasoning
- Optional **alternatives** — e.g., fallback tools to try instead
- For retryable errors, optional retry guidance (e.g., a suggested retry delay)

Implementation pattern: catch the exception, build this metadata object, then return it as the tool result content with `isError: true`.

## Local Recovery Before Escalation

- Analogy: a junior team member should attempt reasonable troubleshooting before escalating to their manager, not escalate immediately at the first sign of trouble
- **Pattern:** when a sub-agent's tool call fails with a transient error, the sub-agent should attempt one or two **local retries** before propagating the failure up to the coordinator
- Escalation flow: tool execution fails → sub-agent catches error and retries locally for transient issues → if retries are exhausted, escalate a **structured** error to the coordinator → coordinator decides next steps (retry differently, invoke another tool/sub-agent, or escalate to a human)
- **Exam distractor to reject:** "terminate the entire pipeline on a single tool failure" — this is a rigid all-or-nothing anti-pattern. The correct behavior is local recovery first, coordinator involvement only once local options are exhausted.

## Tool Overload Problem

- Giving an agent more tools is **not** automatically better — at inference time the model must choose among all available tools, and with too many (especially overlapping) options, **selection reliability degrades**
- **Fix — scoped tool access:** each agent/sub-agent should only see the tools relevant to its specific role (principle of least privilege applied to tool visibility), not a large shared pool
- Example role-based split: a customer-verification sub-agent gets `get_customer`/`escalate`; an order sub-agent gets `lookup_order`/`process_refund`; a communication sub-agent gets `send_email`/`create_ticket` — no overlap, no irrelevant tools
- **Exam pattern:** "which tool configuration best improves [some specialized] agent's accuracy?" → give it a **narrow, purpose-built tool** aligned to its role, not broader access or more tools

## Controlling Execution via `tool_choice`

Three values, each with a distinct exam meaning:

- **`auto`** — the model decides whether and which tool to call (may return plain text instead)
- **`any`** — the model **must** call some tool, but picks which one; use when you need **guaranteed structured output** and plain text is not acceptable
- **`tool`** (specifying a name) — the model **must** call that exact, named tool; use for enforcing a specific extraction schema

**Exam signal:** "guaranteed structured output generation" → answer is `tool_choice: "any"` (not `"auto"`, which still allows the model to skip tool use entirely).

## How It All Fits Together

A robust agent execution flow combines:
1. **`tool_choice`** — forces or permits tool execution as needed
2. **Scoped `allowedTools`** — guarantees correct tool selection by limiting the visible set
3. **Structured error handling (`isError` + `errorCategory` + `isRetryable`)** — lets the coordinator reason about recovery
4. **Local recovery first** — sub-agents retry transient failures locally before escalating
5. **Coordinator escalation** — intelligent routing only when local recovery is exhausted

## Cheat Sheet (3 things to know cold for the exam)

1. The complete structured error signal = `isError` + `errorCategory` + `isRetryable` (+ optional alternatives/description)
2. Scoped tool access improves reliability — adding more tools actively degrades selection accuracy
3. `tool_choice: "any"` is required to guarantee structured output generation

**Anti-patterns to always reject in exam options:**
- Silently returning empty results on tool failure
- Terminating the entire pipeline on a single untried/unretried failure
- Giving agents broad/unscoped tool access "just in case"
