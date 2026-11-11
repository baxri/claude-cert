# Mastering Few-Shot Prompting for Enhanced Language Model Performance

Exam-critical notes from the video transcript (Domain 4 — Prompt Engineering & Structured Output, syllabus task 4.2). Builds directly on the previous lesson (categorical criteria / task 4.1).

## Why Detailed Instructions Alone Still Fall Short

- Instructions describe **what you want**; they don't show **what it actually looks like**
- Even a very long, detailed prompt (hundreds of lines of explicit rules) leaves the model to resolve ambiguity on its own — e.g., a code-review agent might correctly identify a bug but inconsistently label it "critical" vs. "high" across runs, because it was never shown a concrete boundary case
- **The instruction ceiling:** adding more words creates more surface area for ambiguity, not less. A bloated, essay-like prompt with vague words (e.g., "severe") risks the model calibrating against its general training data rather than your project's specific label scheme.
- **Fix — demonstration beats description.** Analogy: reading a lengthy article about a workout vs. watching a demonstration video — the demonstration conveys exact form/positioning that prose alone struggles to specify.
- **Exam claim:** few-shot examples are **explicitly more effective** than long prose passages, because they demonstrate the desired reasoning and output rather than just describing it.

## Anatomy of a Good Few-Shot Example

Each example needs three parts:
1. **Input** — the scenario/query
2. **Reasoning** — *why* this input leads to that output (the derivation)
3. **Output/label** — *what* the correct result is

- **Output-only examples are a trap:** if you give input→output pairs with no reasoning, the model just does **pattern matching** — it looks for which example a new input "resembles" most closely, rather than learning the underlying rule. This breaks down on novel inputs that don't closely resemble any given example.
- **Reasoning-backed examples generalize:** the specific output is scenario-specific, but the *reasoning path* behind it is general — providing reasoning gives the model a transferable decision rule, not just a lookup table.
- Analogy to overfitting in ML: examples without reasoning are like an overfit model — high match accuracy on familiar patterns, poor generalization to new/different inputs. Examples with reasoning let the model extract the underlying decision rule and apply it to novel inputs (like learning a general decision surface instead of memorizing a single point).

## The 2-4 Example Rule (high-yield exam fact)

- **Ideal few-shot example count: 2 to 4 targeted examples**, each with full reasoning, covering a **variety** of cases (not just the obvious ones)
- **Why 5-8+ examples fails:** more examples re-introduces the exact problem you're trying to avoid — it triggers pattern matching and bloats the context, pushing the model back toward "which example does this most resemble" instead of applying a generalized rule
- **Exam trap:** an option like "add six to eight examples" is wrong — it causes pattern matching and context bloat. The correct choice is always 2-4 targeted examples with reasoning.

### Diagnostic Workflow for Choosing Examples

- Run the same task on ~10 inputs, 3x each, and look for **drift** — where does the output flip/become inconsistent across runs?
- Use those **drifting/ambiguous boundary cases** as your few-shot examples, not the obvious cases the model already handles correctly
- This is the inverse of test-driven development: find where the model actually struggles, then write targeted examples for exactly those fault lines — don't waste your 2-4 example budget on cases the model already gets right.

## Example Scenario 1: CI/CD Severity Classification

- Three reasoning-backed examples mapping the severity boundary (critical / high / medium), each showing: the code/scenario, the reasoning for why it falls in that severity, and the label
- E.g., critical example reasoning: "authentication check exists but is silently bypassed — any request proceeds regardless of token validity" → labeled critical
- This resolves the inconsistent-severity-labeling problem directly by anchoring the boundary with concrete, reasoned examples.

## Example Scenario 2: Structured Data Extraction

- Content can arrive as **structured** (e.g., a table) or **unstructured** (e.g., prose) — a single example won't cover both, so give examples spanning different document structures
- Example 1: structured table format — reasoning: "explicit table, values directly stated, no inference needed" → extracts fields directly
- Example 2: prose paragraph format — reasoning: "no explicit table; contract value is stated in prose as $X; payment terms stated as 'within 30 days'" → extracts the same fields, but via inference from prose
- **Exam-relevant:** this scenario (structured data extraction with varied document formats) is one of the official exam scenarios — know how few-shot examples handle multiple input formats for the same extraction task.

## Example Scenario 3: Customer Support Resolve vs. Escalate

- Ambiguous decision boundary: resolve directly vs. escalate to a human
- Fixing inconsistent escalation behavior required **two targeted examples straddling the ambiguous boundary** — one clearly resolvable, one clearly escalate-worthy, each with explicit reasoning
- Example (resolve): "My invoice shows an extra charge for last month, I've been a customer for 4 years" → reasoning: billing discrepancy + long tenure = high trust context, amount likely small, no prior service failure → action: resolve directly (apply credit, apologize, close ticket)
- Example (escalate): "I've been charged three times for the same order and nobody has fixed this in 2 weeks" → reasoning: repeated billing failure + failed prior support contact, not a simple discrepancy → action: escalate

## Sequencing Rule: Tool Description Before Few-Shot Examples

- If you're working with an **agent that has tools**, the tool description must be correct **first** — if Claude doesn't select the right tool to begin with, few-shot examples about reasoning/labeling become irrelevant
- Order of operations: (1) ensure Claude picks the correct tool, (2) then few-shot examples with reasoning guide *how it decides* using that tool
- For plain prompting (no tools/agents involved), few-shot prompting is the primary lever on its own.

## Sample Exam Question (Task 4.2)

**Q:** A CI/CD agent inconsistently assigns severity levels to the same bug class. Most effective fix?
**A:** Add 2-4 targeted few-shot examples with reasoning.
- Reject: "add detailed prose definitions" (helpful but insufficient alone — demonstration beats description)
- Reject: "lower the temperature" (controls randomness, doesn't fix a judgment/calibration problem)
- Reject: "add 6-8 examples" (causes pattern matching and context bloat)

## Cheat Sheet

| Fact | Detail |
|---|---|
| Ideal example count | 2-4 targeted examples |
| Why more examples fail | 5-8+ triggers pattern matching and bloats context vs. teaching a generalized rule |
| Few-shot vs. prose | Few-shot is explicitly more effective than long prose instructions alone |
| Example structure | Input + Reasoning + Output (never output-only) |
| Targeting strategy | Focus examples on ambiguous/boundary cases, found via a drift-diagnostic run, not obvious cases |
| Sequencing with agents/tools | Fix tool descriptions first, then add few-shot reasoning examples |
