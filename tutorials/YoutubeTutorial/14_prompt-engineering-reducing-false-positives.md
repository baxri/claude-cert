# Mastering Prompt Engineering to Reduce False Positives in AI-Powered CI/CD Pipelines

Exam-critical notes from the video transcript (Domain 4 — Prompt Engineering & Structured Output, maps to syllabus task 4.1). First of a multi-lesson arc on prompt engineering.

## The Scenario: Alert Fatigue → Real Vulnerability Missed

- A Claude-powered CI/CD code reviewer flags ~47-50 issues per PR — most are noise (style opinions, naming preferences, theoretical edge cases)
- After a couple weeks, developers learn to ignore the bot entirely and merge without reading the findings
- Eventually a **real SQL injection vulnerability** slips through because it was buried at position 23 in a 50-item noisy list nobody was reading anymore
- **Core lesson:** trust is holistic — you cannot have a bot that's 90% noise in one category and expect developers to still trust its other categories. Once dismissed, everything gets dismissed (the "boy who cried wolf" problem).

## Fix Roadmap (4 steps)

1. Don't write vague instructions
2. Confidence-score filtering — **also doesn't work** (explained below)
3. **Categorical criteria** — the actual fix
4. Separate concerns across pipeline steps rather than cramming everything into one prompt

## Step 1: Vague Instructions Don't Work

- Example failing prompt: *"Be thorough and conservative in your analysis."*
- Sounds reasonable to a human, but the model has no calibration for what "conservative" means in **your specific context** — it produces everything: style comments, opinionated naming, micro-optimizations, irrelevant suggestions
- **Exam fact:** vague adjective-based instructions ("be conservative," "flag only high-confidence issues") **do not reliably improve precision** — the model can't calibrate an adjective to your team's standards.

## Step 2: Confidence-Score Filtering Also Fails

- Example failing prompt: *"Only flag things you're 80% confident about."*
- Problem: the model's confidence is calibrated to its **training distribution**, not your codebase or your team's definition of "a real problem." It might be 85% confident about a naming pattern (common in training data) and only 65% confident about a genuine SQL injection — confidence ≠ severity or relevance to you
- You're asking the model to judge its own judgment — unreliable, same as trusting a developer's own "yes it'll work" instead of independent testing
- **Exam fact:** confidence-threshold instructions also fail to improve precision, for a different reason than vague adjectives (miscalibration to training data, not to your standards).

## Step 3: The Real Fix — Categorical Criteria ("Stop Asking, Start Telling")

- Both prior approaches ask the model to **develop judgment** about what matters — but the model doesn't have your team's judgment, only its training data
- **The fix is not better judgment instructions — it's explicit, categorical rules.** Don't ask the model to decide; command it.

### Anatomy of a Categorical-Criteria Prompt

Replace vague adjectives with **explicit named lists**:
- **Report — Critical:** e.g., known security vulnerabilities (SQL injection, XSS, CSRF)
- **Report — Error:** significant logic errors, missing error handlers in critical paths, race conditions in concurrency blocks
- **Report — Security:** hard-coded credentials, weak encryption implementations
- **Never report (explicit negative constraints):** linter-approved style/formatting, naming convention choices (e.g., snake_case), sub-5ms performance optimizations, minor stylistic nitpicks that don't affect functionality or security
- **Output format:** specify the exact machine-readable shape required (e.g., "output must be a JSON array of objects with these fields") — critical for CI/CD pipelines where the next step needs to parse the result

### Before/After Result

- Before (vague prompt): ~47 flagged issues, mostly false positives, real vulnerability buried
- After (categorical criteria): 3 flagged issues — SQL injection (critical), missing error handler (high), race condition (high) — all real, all actionable
- **Framing:** this isn't "less aggressive reviewing," it's **better-defined reviewing**.

## Going Deeper: Explicit Severity Levels with Code Examples

- Don't just describe severity levels in words — **show code examples of what each severity level looks like**, paired with reasoning
- This previews the next lesson's topic (few-shot prompting) — showing reasoning-annotated examples teaches the model your team's actual standard, not just a category label
- **Exam fact:** explicit severity criteria with code examples beats a longer, more "descriptive" prompt. A longer prompt without concrete examples can even confuse the model — descriptive AND concise, backed by examples, beats sheer length.

## The Nuclear Option: Temporarily Disable Noisy Categories

- If one whole category (e.g., performance suggestions) has an 80% false-positive rate and tightening its criteria hasn't worked, the counterintuitive but correct move is to **temporarily disable that category entirely** — not just lower its threshold
- Rebuild trust with the categories that are working, then reintroduce the disabled category later with better criteria and real examples (few-shot)
- **Exam trap:** if an exam option says "lower the threshold for the noisy category," that's wrong — a lower threshold on an already-miscalibrated category still produces false positives. The correct answer when offered is **disable the category temporarily**, not adjust its threshold.

## Same Pattern, Different Arena: Structured Data Extraction (Scenario 6 preview)

- In a data-extraction pipeline (e.g., pulling structured fields from medical reports, insurance documents, financial filings), the "false positive" problem shows up as **fabricated data** — the model fills in fields that don't actually appear in the source document (a patient ID, an inferred diagnosis code, even a suggested medication dosage that wasn't in the source)
- **Root cause:** the prompt said "extract structured data" but never told the model what to do about missing information — so the model fills the gap using its own judgment. **This is a prompt failure, not a model failure.**
- **Fix — explicit extract/skip rules:** e.g., "Always extract patient ID and ICD-10 codes exactly as written. Never fabricate. If a field is absent, return `null`. If a value is ambiguous, return the raw text as-is." Specify precisely what to do for missing vs. ambiguous data.
- **Stakes:** "null is a valid answer; a fabricated ID is a lawsuit" — in domains like medical/financial data, hallucinated fields are not a cosmetic issue, they're a serious liability. You must explicitly define what "accuracy" means in your domain within the prompt.

## Exam Prep Summary (Task 4.1)

1. Vague confidence instructions ("be conservative," "flag high-confidence issues") do not reliably improve precision — the model can't calibrate an adjective/threshold to your standards
2. **Categorical criteria** = specific named reports + explicit skip rules, not general thresholds — this is the answer to "most effective first step to reduce false positives"
3. Trust collapse is **cross-category** (holistic) — fix a noisy category by **disabling it temporarily**, not by lowering its threshold
4. Few-shot examples are the *next* refinement layer (covered in the following lesson) — don't confuse "what's the first fix" (categorical criteria) with "how do you improve further" (few-shot examples with reasoning)

## Cheat Sheet

| Symptom / Question | Wrong Answer | Right Answer |
|---|---|---|
| Reduce false positives, first step | "Be conservative" / vague adjectives | Categorical criteria (explicit named report/skip rules) |
| Reduce false positives further | Raise confidence threshold | Few-shot examples with reasoning (next lesson) |
| One category is consistently noisy | Lower its threshold | Temporarily disable that category |
| Extraction pipeline fabricates missing fields | Trust the model's inference | Explicit null/raw-text rules for absent/ambiguous data |
| CI/CD next step needs parseable output | Leave format unspecified | Explicitly specify required output shape (e.g., JSON array with named fields) |
