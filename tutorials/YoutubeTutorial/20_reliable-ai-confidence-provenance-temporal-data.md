# Building Reliable AI Systems: Accuracy, Confidence, Provenance, and Temporal Data

Exam-critical notes from the video transcript (Domain 5 — Context Management & Reliability). Final episode of the core lesson series, covering human-review workflows and information provenance in depth.

## The 97% Trap: Aggregate Accuracy Hides Category Failures

- A document-extraction system tests at 97% overall accuracy and ships — months later, finance reports numbers don't add up
- Root cause: the system was near-perfect on standard invoices (the bulk of volume) but only ~60% accurate on handwritten receipts — a small slice of volume, but a high-stakes exception category
- Analogy: a hospital's overall 95% recovery rate can hide a 60% recovery rate in a specific department (e.g., cardiac) — you need failure rates broken out by category, not just the average.

### Why This Happens

1. **High-volume cases dominate the average** — if 90% of documents are simple and the AI nails them, the aggregate looks great even if the remaining 10% fails badly
2. **Critical categories are often low-volume but high-stakes** — the exception cases that matter most are exactly the ones an aggregate metric buries
3. **Errors on outlier types compound over time and are systematic, not random noise** — once an AI system consistently fails on a specific document type, that's a structural failure requiring a structural fix, not something to dismiss as statistical noise at a high overall accuracy rate.

## Stratified Sampling — Review Smarter, Not Everything

- You can't manually review every document, but you can review the *right* representative sample from each category
- Analogy: a car factory doesn't inspect every car off the line — it samples across every assembly line, shift, and model, so a defect spike in one line gets caught fast.

### The 4-Step Process

1. **Categorize** — split incoming documents into distinct buckets/strata (e.g., standard PDFs, scans, handwritten forms, multi-language contracts)
2. **Sample per stratum** — pull a representative sample from *each* category, even if a category (e.g., handwritten forms) is only ~5% of total volume — low-volume categories still get their own dedicated sample
3. **Review and measure per category** — human reviewers check each batch, producing **accuracy by category**, not just an overall average
4. **Prioritize action** — if one category (e.g., handwritten forms) shows a low accuracy rate (e.g., 60%), route all documents in that category to human reviewers immediately rather than waiting for the aggregate metric to reflect the problem

**Exam fact:** stratified random sampling is the correct architectural answer for measuring error rates without reviewing everything — it gives maximum diagnostic signal for minimum review effort.

## Field-Level Confidence Scores — "Know What You Don't Know"

- The right question isn't "is this document correct?" — it's "is this **specific field** correct?" Confidence lives at the field level, not the document level.
- Analogy: a good weather forecast doesn't say "80% chance of rain" for an entire region — it gives localized, specific probabilities per area, which is what makes it actually useful.

### Example Pattern

Each extracted field carries its own confidence score and source location, e.g.:
- `invoice_number`: high confidence (clearly labeled, top-right header) → auto-route
- `total_amount`: high confidence (clear table row) → auto-route
- `vendor_address`: low confidence (unreadable handwritten string) → **flag for human review** — only this specific field, not the whole document

- **Calibration matters:** confidence scores must be calibrated using **labeled validation sets** (real examples with known ground truth) — an uncalibrated model's raw self-reported confidence is just a guess, not a reliable signal.

## Routing: When the AI Knows It Doesn't Know

- Analogy: autopilot handles most of a flight, but hands control back to the pilot during turbulence, storms, or landing — the pilot's most important job is exactly those edge cases.

### Two Valid Routing Triggers to Human Review

1. **Low field-level confidence** — a field's confidence falls below a set threshold (e.g., 75-85%) → route to human review queue
2. **Contradictory fields within a document** — e.g., a total that doesn't match the sum of line items, or an internally inconsistent date → flag for manual review, since internal inconsistency itself signals unreliable extraction

### Two Anti-Patterns to Avoid (exam-tested)

1. **Silent discard** — dropping low-confidence fields creates silent gaps; downstream systems have no idea they're working with incomplete data
2. **Force-through** — letting low-confidence extractions pass without flagging contaminates downstream data with unaudited, unreviewed (possibly hallucinated) values

## The Attribution Problem: "Where Did That Number Come From?"

- In a multi-agent pipeline (search agent → analysis agent → synthesis agent → report agent), a final claim like "adoption increased 47% year-over-year" is worthless if nobody can trace which source paper/methodology it came from
- **Fix — claim-source mappings** (introduced earlier in the series, in the context of passing structured context between sub-agents): every claim carries its source attached, the same way a passport travels with you across every border crossing. This lesson extends that idea to what happens when the mapping **survives or breaks** as it moves through human review and downstream validation.
- A claim-source mapping should include the claim text, the source document, section/location, publication date, and a confidence score — enough that a human reviewer (or the coordinator) can independently verify it.

### When Sources Disagree

- If two sources report conflicting values (e.g., market size reported as $12B by one source, $18B by another, using different methodologies), **do not**:
  - Pick the highest value
  - Pick the lowest value
  - Average them (this fabricates a third, unsupported number)
- **Correct approach:** annotate **both** findings with their respective sources, surface the conflict to the coordinator, and escalate to a human if the coordinator can't resolve it independently. Never silently resolve a source conflict by picking or blending values.

## Temporal Data: "A Fact from 2019 Is Not a 2024 Fact"

- Data has an implicit "expiration" — what was true/best-practice years ago may no longer be current (analogous to a food product's best-by date: technically still edible in some cases, but not safe to treat as fresh)
- **Fix:** require two date fields in every structured output:
  - **Publication date** — when the underlying information was originally published/measured
  - **Collection date** — when the agent actually retrieved/extracted the data
- **Exam fact:** requiring publication and collection dates in structured outputs to prevent temporal misinterpretation is a **structural schema fix**, not a prompt-engineering or retry-logic fix — reject any exam option suggesting better prompting or retries as the solution to stale/temporal data problems.

## Putting It Together — Scenario 6 (Structured Extraction)

A robust extraction pipeline combines all of the above:
1. **Field-level confidence + source region** — both required (non-optional) fields in the output schema
2. **Stratified sampling** — categorize documents (standard PDFs, scans, handwritten, etc.), sample and track accuracy per category, not in aggregate
3. **Routing logic** — low confidence or contradictory fields → flag to coordinator → human reviewer
4. **Mandatory date fields** — publication date and collection date on every record
5. **Claim-source mappings** — preserved throughout, so any downstream consumer can trace a value back to its origin

## Scenario 3 — Provenance Across Agent Boundaries (Multi-Agent Research)

Tracing responsibility through a 4-agent research pipeline:

1. **Search agent** — must pass forward the URL, publication date, and domain credibility with every claim it finds. This is often the **first point attribution gets dropped** — don't let it happen here.
2. **Analysis agent** — reads/extracts from sources; must preserve page numbers and section headings alongside the content, carrying forward the metadata handed off by the search agent.
3. **Synthesis agent** — **the highest-risk point for provenance loss.** When combining claims from multiple upstream sources, it must keep all source tags attached to their respective claims — it must never collapse multiple sources into one unified, untraceable claim.
4. **Report agent** — writes the final output, including all citations passed forward from the synthesis agent, so the end consumer can trace every claim back to its original source.

**Exam call-out:** the synthesis agent is explicitly the highest-risk point for provenance loss — its prompt and output schema must explicitly require source tags to survive the combination step.

## Sample Exam Question

**Q:** A structured extraction system processes 10,000 documents/day at 96% overall accuracy, with document-level confidence above 80% auto-routing to downstream systems — but errors persist specifically in legal contracts. What's the most likely root cause?
**A:** Document-level confidence masks category-specific failures; requires stratified sampling. (Reject: "the global 80% threshold is too low" — confidence should operate at the field level, not the document level; "errors are statistically expected at 96%" — dismisses a systematic category failure as noise; "needs more few-shot examples" — this is a data/categorization/confidence problem, not a prompt-engineering problem.)

## Cheat Sheet

| Concept | Key fact |
|---|---|
| Aggregate accuracy | Can mask much lower accuracy in specific high-stakes categories (e.g., 97% overall vs. 60% on one document type) |
| Stratified sampling | Measures error rates per category without reviewing everything — the correct fix for category-specific failures |
| Confidence scores | Belong at the **field level**, not the document level; must be calibrated against labeled validation sets |
| Routing triggers | Low field confidence, or contradictory fields within a document → human review queue |
| Routing anti-patterns | Silent discard of low-confidence fields; forcing low-confidence data through unflagged |
| Claim-source mappings | Every claim must carry its source; conflicting sources get annotated (both) and escalated, never averaged or arbitrarily chosen |
| Highest provenance-loss risk point | The synthesis agent, when merging claims from multiple sources |
| Temporal data | Require publication date + collection date in every structured output — a schema fix, not a prompting fix |
