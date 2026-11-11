# Leveraging Claude Code: Plan Mode, Direct Execution, and Efficient AI-Assisted Development

Exam-critical notes from the video transcript (Domain 3 — Claude Code Configuration & Workflows, syllabus task 3.4).

## The Scenario: Blind Refactor Gone Wrong

- Asking Claude to "refactor this 40,000-line monolith into microservices" with no plan phase can leave a half-refactored codebase with broken imports and failing tests
- **Root lesson:** not all tasks are the same — some need Claude to think before acting (**plan mode**), others Claude can just execute (**direct execution**). Knowing which to use is the skill.

## Direct Execution

- The default mode: give Claude a task, it does it — no planning ceremony
- **Use when:** you already know exactly what needs to happen, scope is clear, output is predictable, and mistakes are cheap to fix
- Good fits: writing unit tests for an existing class/function, fixing a bug with a clear stack trace, updating config files, generating boilerplate
- **Common thread:** the destination is known — little ambiguity about what "correct" looks like

## Plan Mode

- An **investigation phase before any code is written**: Claude reads broadly across the codebase, understands current architecture, identifies dependencies, considers multiple approaches/trade-offs, and presents a plan for approval before executing
- **Why it beats direct execution for big changes:** in direct execution on a large/ambiguous task, Claude works file-by-file, hopping to dependencies as it discovers them — if the approach turns out wrong, reversing it (without version control) means manually undoing work, burning tokens
- Also valuable for **greenfield projects** — giving proper requirements up front in plan mode can produce a better architectural foundation than jumping straight to boilerplate
- **Exam triggers:** large-scale architectural changes, multiple valid approaches, refactor/migrate/re-architect scenarios → **plan mode**

### Decision Tree: Plan Mode vs. Direct Execution

Go with **plan mode** if any of these are true:
- Change touches **more than one file non-trivially** and there's ambiguity about how
- **Multiple valid approaches** exist and you're not sure which is best
- The change would be **expensive to reverse** if wrong
- The change **alters the architectural shape** of the system

Otherwise → **direct execution**.

**Common mistake:** defaulting to direct execution because it feels easier/faster — this trades a few minutes of upfront planning for token waste and a lower-quality result on tasks that actually needed a plan.

### Sample Exam Question

**Q:** Migrating a highly coupled Node.js monolith with significant interdependencies — what do you do?
**A:** Use plan mode to generate a phased strategy before making changes. (Reject: guessing dependencies and migrating one service at a time; building extensive test suites for all services before any migration — better handled by having Claude review *existing* tests in plan mode and plan the migration/expansion of the test suite alongside it; running an autonomous migration script with no user input.)

### Plan Mode and Direct Execution Are Not Rivals

They form a **sequential workflow**: use plan mode where needed to decide the approach, then use direct execution to carry out the approved plan. Insert plan mode only where the decision criteria above apply; skip it otherwise.

## Practical Demo Notes (model/effort selection)

- For simpler scaffolding tasks (e.g., generating an Angular app blueprint), **Sonnet is sufficient** — no need to default to Opus and burn more tokens/credits
- Use **`/model`** to check/switch the current model, and arrow keys to adjust **effort level** (low ↔ high/max) — increase effort/model tier when a session gets stuck in unproductive loops (e.g., Sonnet at medium effort couldn't resolve a persistent runtime error; switching to Opus at high effort resolved it)
- Claude Code (terminal) and Cline (VS Code extension) both work as clients; Cline has a separate credits system unrelated to a Claude Pro subscription — an "insufficient balance" error from Cline is independent of your Claude subscription status
- Always **review what Claude is about to do** rather than blindly approving — Claude requests consent for actions, and you retain the ability to reject a step at any point

## The Explore Sub-Agent

- **Problem:** on a large, real-world codebase, asking Claude to "understand this complex system before planning a change" causes it to read files, follow import chains, read tests — filling the context window before any actual reasoning/planning happens (**context window exhaustion**)
- **Fix:** the **Explore sub-agent** — a dedicated sub-agent whose only job is codebase discovery. It reads broadly, forms an understanding, and returns a **compact summary** back to the main session, keeping the main session's context intact
- Invocation is just prompting — no special configuration/command: e.g., "use the Explore sub-agent to give me a summary of this project"
- **Exam fact:** whenever a question mentions **context window exhaustion** during codebase discovery on a large project, the answer is the **Explore sub-agent**.

## Iterative Refinement Patterns (explicitly tested on the exam)

Three techniques for pushing Claude's output from "good enough" toward near-perfect:

### 1. Feedback Loop
- Claude produces output → you review → you give **specific** feedback → Claude revises
- "Make it better" is useless feedback — it gives Claude nothing concrete to act on
- Good feedback names the exact problem (e.g., "the try/catch on line X needs to log the full error object")

### 2. Test-Driven Iteration
- Give Claude the tests first, let it generate code, run the tests, and feed back failures
- Removes ambiguity about what "success" means — Claude iterates with precision when it knows exactly which assertions must pass (machine-generated feedback, not subjective judgment)

### 3. Concrete Examples
- "One concrete snippet beats a page of prose" — provide before/after transformation examples (few-shot style) rather than long prose descriptions of what you want
- Combines with test-driven iteration: feeding failing test output/logs back to Claude as a concrete example of what needs fixing

## The Interview Pattern

- Inverts the usual flow: instead of trying to write the perfect prompt yourself, brief Claude on the high-level goal and explicitly ask it to **ask you clarifying questions before starting**
- Rationale: Claude often knows better than you do which missing detail would most affect the output — you might spend a long time writing an elaborate prompt and still miss something Claude would have asked about directly
- Example prompt: "Build me a REST API for user management. Before you write any code, ask me clarifying questions about requirements, constraints, or preferences that would significantly change your approach."
- **Exam signal:** "reducing rework" / "improving first-pass quality" / "unclear scope" / "multiple possible iterations" in an open-ended implementation task → **interview pattern**
- **Not mutually exclusive with plan mode** — the interview pattern is about *how you prompt*, and can be combined with plan mode (ask clarifying questions first, then let Claude produce the plan, then execute).

## Cheat Sheet

| Signal in the question | Answer |
|---|---|
| Large-scale refactor, hard to reverse, architectural impact | Plan mode |
| Clear stack trace, obvious single-file fix | Direct execution |
| Context window filling up during large codebase discovery | Explore sub-agent |
| Ambiguous requirements / transformation task, reduce rework | Interview pattern |
| Need precise, unambiguous success criteria for iteration | Test-driven iteration + concrete examples |

**Sample question:** "Extract DB queries from a 200-file monolith to an ORM" → multiple files, hard to reverse, architectural impact → **plan mode**.
