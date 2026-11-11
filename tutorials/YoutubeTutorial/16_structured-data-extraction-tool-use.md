# Production-Grade Structured Data Extraction with Agent Tool Use and Schema Enforcement

Exam-critical notes from the video transcript (Domain 4 — Prompt Engineering & Structured Output, syllabus task 4.3/4.4 area). Builds on tool design (Domain 2) and prior prompt-engineering lessons.

## Why Free-Form Prompting Breaks Production Pipelines

Three concrete failure modes when you just ask a model for "JSON" via prose instead of enforcing structure:

1. **Markdown wrapping** — the model wraps output in ```` ```json ... ``` ```` triple backticks; a strict parser expecting raw JSON hits a backtick instead of a bracket and throws a fatal syntax error. The data itself may be correct, but the *structure* of the response isn't.
2. **Schema drift** — on complex documents, the model omits optional fields, renames keys, or alters nesting inconsistently across runs, collapsing downstream reliability.
3. **Fabrication of required fields** — when a field's data isn't present in the source, the model may invent a plausible-sounding value rather than admit absence. Unacceptable for sensitive domains (e.g., patient data).

## The Fake Tool Paradigm (core technique)

- Define a tool whose only job is to **enforce a schema** on the model's output — it contains no real business logic, it just declares an `input_schema` (JSON Schema) and forces Claude to populate it via a `tool_use` block
- Example: an `extract_invoice_data` tool with properties like `vendor_name`, `total_amount`, `invoice_date` — Claude must call this tool with data matching the schema, rather than freely generating prose or loosely-structured JSON
- This converts structured extraction from "ask nicely for JSON" into a strongly-typed contract enforced by the tool-calling mechanism itself
- **Exam framing:** `tool_use` + JSON Schema is the most reliable/"apex" structured output mechanism — prefer it over prompt-based JSON requests.

## `tool_choice` Revisited (now specifically for extraction)

| Value | Behavior | Fit for extraction |
|---|---|---|
| `auto` | Claude may call a tool or respond with plain text | **Unreliable** for strict extraction pipelines — Claude might just not call the tool. Better suited to conversational/general agents. |
| `any` | Claude must call **some** tool, but picks which | Guarantees a `tool_use` block every time — good default for extraction when tool selection is already reliable (via good descriptions) |
| `{"type": "tool", "name": "..."}` | Claude must call **that specific** tool | Maximum lockdown — best when you know exactly one tool must fire every time (e.g., a single-purpose extraction pipeline) |

## Required + Nullable: Preventing Fabrication (high-yield exam fact)

- **The trap:** marking a field `required` with type `string` (and no null option) implicitly pressures the model to produce *some* value even when the source data doesn't contain it — risking fabrication/hallucination
- **The fix:** make the field **required AND nullable** (e.g., type `["string", "null"]`) — the field is guaranteed to be *present* in the output, but `null` is an explicit, valid escape hatch when the source data doesn't support a real value
- **Null is a valid, structurally correct value — a fabricated value is not.** This is the central principle of the lesson.

## Syntax vs. Semantics — Two Different Failure Classes

| | Syntax / structural failure | Semantic / interpretation failure |
|---|---|---|
| Examples | Missing required fields, string instead of number, malformed JSON | Reading the wrong table row, confusing similar terms, hallucinated values |
| What fixes it | Strict JSON Schema + `tool_use` enforcement | Business logic validation, cross-field consistency checks, human review (e.g., in a post-tool hook) |

- **Critical distinction:** a schema guarantees your output is *structurally flawless* — it guarantees **nothing about the truth of the data**. A model can extract a due date when asked for an invoice date, and a schema-conformant parser will happily accept the wrong value.
- **Exam pattern:** "extraction system passes JSON schema validation 100% of the time but occasionally has wrong values because the model read the wrong table row" → this is a **semantic** error, not a structural one. The fix is **adding semantic validation (business rule checks, cross-field consistency)** — stricter schemas or more retries cannot fix a misinterpretation.

## The Self-Correcting Retry Loop

- Pattern: run extraction → validate structurally → if invalid, append the error as feedback into the conversation (assistant/tool-result flow) and retry, up to a max retry count
- **Retries are format correctors, not data generators.** If a field is missing from the source document, retrying will never manufacture data that doesn't exist — retries should only be used to fix *structural/format* mismatches (e.g., wrong date format), not to pressure the model into inventing missing information.
- If the underlying data is genuinely absent, that's not something a retry loop can or should fix — accept `null`, don't force generation.

### Sample Exam Scenario

**Q:** An extraction pipeline fails validation on every retry because a required field (e.g., tax ID) is genuinely missing from the documents being processed. What's the correct architectural response?
**A:** Change the field's type to allow `null` (required + nullable) — this stops the retry loop from failing indefinitely and stops pressuring the model to hallucinate. (Reject: adding more retries — will never find data that isn't there; removing the field entirely — loses the guarantee that the field is at least present/accounted for.)

## Putting It Together: Targeted Retry Loop Design

For a production extraction pipeline (e.g., inside a customer support agent's tool chain):
1. **`tool_choice: "any"`** — guarantee Claude calls a tool every time (assuming tool descriptions are already reliable)
2. **Required + nullable schema fields** — structural integrity without forced fabrication
3. **Retry loop for structural/format failures only** — bounded by a max-retry count, feeding the specific error back into the conversation
4. **Semantic validation layer** (e.g., in a post-tool hook) — business logic and cross-field consistency checks, separate from schema validation
5. **Human handoff** — once max retries are exhausted or a semantic failure can't be auto-resolved, escalate rather than looping indefinitely or fabricating a result

## Cheat Sheet

| Fact | Detail |
|---|---|
| Best structured output mechanism | `tool_use` + JSON Schema — preferred over prompt-based "please respond in JSON" |
| Guaranteeing tool execution | `tool_choice: "any"` (any tool) or `{"type": "tool", "name": "..."}` (specific tool) |
| Preventing fabrication | Required + nullable fields — `null` is a valid, honest value |
| Schema validation fixes | Syntax/structural errors only (missing fields, wrong types, malformed JSON) |
| Schema validation does NOT fix | Semantic errors (wrong table row read, hallucinated/incorrect values) — needs business logic / cross-field validation |
| Retry loops fix | Format/structure failures only — never a substitute for missing source data |
| Missing data at max retries | Escalate to human handoff, don't keep retrying or force generation |
