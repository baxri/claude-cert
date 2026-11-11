# Enhancing Reliability in Multi-Agent Systems and Claude Code Workflows

Exam-critical notes from the video transcript (Domain 5 — Context Management & Reliability). Second-to-last lesson in the series. Two topics: coordinator-level error propagation, and Claude Code context degradation.

## The Sneakiest Failure Mode: Silent Death

- A scenario: an overnight multi-agent research job produces **no error, no output, no report** by morning — total silence, nothing to debug. This is one of the most painful failure modes in agentic systems.

## Two Different Error-Propagation Layers (don't confuse them)

- **Tool → sub-agent** (covered in an earlier lesson, Domain 2): a single MCP tool signals failure back to its calling agent — the `isError` flag, error categories, retryable logic, all from the *tool developer's* perspective.
- **Sub-agent → coordinator** (this lesson, Domain 5): the whole sub-agent itself fails and must propagate a structured failure up to the coordinator — a different, higher layer of the hierarchy. A tool failure that a sub-agent can't locally resolve becomes the input to *this* layer.

## Why Generic Errors Paralyze Coordinators

- Analogy: a contractor returns from a job site and just says "things didn't work out," no photos, no explanation — you (the coordinator) have no idea whether to retry, get a different contractor, or intervene yourself
- A sub-agent returning something like `{"error": "operation failed"}` leaves the coordinator with **no actionable information** — it can only guess.
- A detailed error (e.g., naming the timeout, whether it's retryable, what was being attempted, and an alternative path) lets the coordinator make an informed decision.

## Four Ingredients of a Recovery-Ready Error Response

1. **Error category** — what kind of failure (timeout, permission, invalid input, policy violation, etc.)
2. **Is it retryable** — a single, load-bearing boolean signaling whether retrying could plausibly help
3. **Attempted query/task** — what the sub-agent was actually doing when it broke, so the coordinator understands the gap, not just that *something* failed
4. **Partial results + alternatives** (the most overlooked ingredient) — what data was collected before failure (e.g., 3 of 5 expected documents retrieved) so it isn't discarded, plus what alternative approaches/sources could be tried if a retry doesn't work

Each of these fields should be concrete, not vague — this is the difference between a coordinator that can act intelligently and one that's stuck guessing.

## Access Failure vs. Valid Empty Result (a critical distinction)

Both can superficially look like "nothing was found," but they mean very different things and demand different `is_retryable` values:

| | Access failure | Valid empty result |
|---|---|---|
| Example | Network timeout, auth error, service down | The query executed successfully, but genuinely matched nothing |
| Analogy | Librarian says "the database is down, come back later" | Librarian says "the database works fine, that book doesn't exist" |
| `is_retryable` | **true** | **false** |

- **Exam fact:** the `error_category` field is what enables the coordinator to distinguish these two cases. Treating them the same either wastes time endlessly retrying queries that will never return results, or gives up on queries that would have succeeded on a second attempt.

## Two Error Anti-Patterns (exam-tested)

1. **Silent suppression** — an exception occurs inside a sub-agent, but it's swallowed and the agent returns a "success" status with empty results anyway. The coordinator has no way to know something actually went wrong, so it confidently accepts an empty result as if it were valid — producing **confidently wrong output**, arguably the worst failure class because it's invisible.
2. **Immediate termination** — a sub-agent failure causes the *entire pipeline* to abort, discarding all work done by other sub-agents, and the coordinator never gets the chance to make a decision.
   - **Fix:** sub-agents should never have the authority to terminate the pipeline themselves. Only the **coordinator** decides whether to terminate, based on structured error information propagated up to it — consistent with the escalation pattern from the previous lesson (a sub-agent proposes/reports, the coordinator decides).

## Local Recovery Before Escalation (reiterated from an earlier lesson)

- Before escalating to the coordinator, a sub-agent should attempt to resolve the problem itself when appropriate:
  - **Transient errors** (timeouts, intermittent network issues) → retry locally a bounded number of times
  - **Wrong format** → adjust the query
  - **Source is down** → try a backup source
- Only once local options are exhausted should the sub-agent escalate to the coordinator with a full structured error response.

### Sample Exam Question

**Q:** A web-search sub-agent returns a generic "search unavailable" status. The coordinator can't tell if it's transient or if partial results exist. What's the most effective action?
**A:** Redesign the sub-agent to return **structured error context** (category, retryable, attempted query, partial results, alternatives). (Reject: blindly retrying with no context, or terminating the pipeline — the anti-pattern covered above.)

---

## Context Degradation in Claude Code Sessions

- Every tool call, file read, and bash command in a Claude Code session gets appended to conversation history — over a long session, the context window fills with raw output, and Claude Code can quietly "forget" early findings (to conserve tokens) even while it keeps running
- Symptoms: early findings get pushed out of the window; if you point out something was missed, Claude re-reads files it already processed (wasted tokens); connections between related files can become disconnected across a long session
- "Just use the most powerful model at max effort" isn't a real fix — it burns through context/cost faster, not slower.

### Three Fixes

#### 1. Scratchpad Files
- A plain markdown/JSON file (not part of conversation history) where Claude Code writes its key findings as it explores a codebase
- When context fills up and old findings would otherwise disappear, Claude Code can **re-read the scratchpad file** to recover what matters instantly, instead of re-exploring the codebase from scratch
- **Catch:** this doesn't happen automatically — you must explicitly instruct Claude Code to maintain a scratchpad (via a direct prompt in an interactive session, via `CLAUDE.md` for consistent/CI usage, or coded into agent logic for programmatic agents)
- Analogy: a detective's case notebook — write findings continuously, reread to reconstruct the full picture later

#### 2. `/compact` Command
- Compresses conversation history into a summary to free up room in the context window
- Can trigger **automatically** around ~90% context usage in an interactive session
- **Exam fact:** `/compact` is a typed command — it **cannot** currently be invoked as a parameter/flag in a CI/CD pipeline (e.g., passed to the `claude` CLI invocation). If an exam question asks whether you can run `/compact` in a CI/CD pipeline, the answer is **no** (as of this material).

#### 3. Manifest File (session checkpointing)
- **Problem:** exploring a large codebase can take hours; if the session/terminal crashes partway through, you lose recent progress and restart from scratch (a summary alone isn't the same as resuming from an exact checkpoint)
- **Fix:** before major steps, Claude Code writes its current state to a **manifest file** at a known, predictable location (inside `.claude/`, alongside skills/commands/scratchpad — keeping everything Claude-related in one predictable place avoids wasted token search time)
- Analogy: a Git commit — checkpoint before risky/time-consuming work, so a crash lets you resume from the last checkpoint instead of zero
- **Manifest vs. scratchpad:** the scratchpad tracks *what's been learned* (findings); the manifest tracks *where progress currently stands* (a checkpoint/state snapshot) — they solve different problems and are complementary, not interchangeable.
- On restart, combine the manifest file (exact checkpoint) with a short summary (context) to resume effectively.

## Making Silent Failures Impossible — Three-Layer Summary

1. **Error layer** — structured error escalation: category, retryable, attempted query, partial results, alternatives
2. **Context layer** — scratchpad files (continuous findings) + `/compact` (context compression) to prevent Claude Code from silently forgetting
3. **Recovery layer** — manifest file for checkpoint-based session recovery after a crash

## Cheat Sheet

| Concept | Key fact |
|---|---|
| Sub-agent → coordinator errors | Must include category, retryable, attempted task, partial results + alternatives |
| Access failure vs. empty result | Access failure → retryable true; valid empty result → retryable false; `error_category` is what distinguishes them |
| Silent suppression | Anti-pattern — swallowing an error and returning a false "success" produces confidently wrong output |
| Immediate termination | Anti-pattern — only the coordinator should have authority to terminate the pipeline |
| Scratchpad file | Persists key findings to disk; must be explicitly triggered, not automatic |
| `/compact` in CI/CD | Not possible — it's a typed interactive command, no CLI parameter equivalent |
| Manifest file | Checkpoints session state at a predictable path (`.claude/`) for crash recovery, distinct from the scratchpad's findings log |
