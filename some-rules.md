# Exam Pattern Takeaways

## Pattern 1 — Inconsistent Subjective Judgment

**Rule:** Drifting subjective judgments (severity/quality/accuracy) → **explicit criteria + concrete examples per level**.

**Reject:**
| Distractor | Why wrong |
|---|---|
| Static lookup table (type → value) | Ignores context, criteria still vague |
| Relative/proportional ranking within batch | Creates cross-batch inconsistency |
| "Include reasoning + manually calibrate" | Doesn't scale, doesn't fix model |
| Upgrade model tier | Bigger model, still no anchor |
| General few-shot without explicit criteria | Teaches mimicry, not the rule |

---

## Pattern 2 — Variable Case-Specific Completeness Gaps

**Rule:** Output is correct but misses *different* things each case → **self-critique step** (draft → evaluate vs checklist → revise).

**Reject:**
| Distractor | Why wrong |
|---|---|
| Few-shot for "common case types" | Can't cover variable long-tail |
| Upgrade model tier | No self-check mechanism added |
| Customer confirmation step | Shifts work to user |
| Add human review | Ruled out by "no human overhead" |

**Fix selector:**
| Failure mode | Fix |
|---|---|
| Format/structure inconsistent | Few-shot (show format) |
| Tool selection on edge cases | Few-shot w/ reasoning |
| Completeness varies case-by-case | Self-critique checklist |
| Subjective judgment drifts | Pattern 1 (criteria + examples per level) |

**CRITICAL — self-critique vs independent review:**
| Task | Use |
|---|---|
| "Is it **correct**?" (bugs/accuracy/reasoning quality) | Independent reviewer |
| "Did I **include** X, Y, Z?" (checklist scan) | Self-critique |
| "Matches required **structure**?" | Self-critique |
| "Addressed **each part** of multi-part input?" | Self-critique |

**Keyword test:** *accuracy / bugs / errors / reasoning quality* → independent reviewer · *completeness / coverage / missing / inconsistent components* → self-critique.

---

## Pattern 3 — Prose Spec Keeps Getting Misinterpreted

**Rule:** Prose specification failed 2+ iterations → **switch medium to concrete I/O examples**. Don't iterate on more precise prose.

**Reject:**
| Distractor | Why wrong |
|---|---|
| Rewrite prose with more technical precision | Same failing medium; prose has irreducible ambiguity |
| JSON schema validation | Verifies *structure*, doesn't teach *transformation logic* |
| Ask Claude to explain its interpretation | Diagnostic only; doesn't fix the spec |

**Keyword test:** *"interpreted differently each time" / "describing in prose" / "doesn't match expectations after iterations"* → concrete I/O examples.

---

## Pattern 4 — When to Escalate to Human

**Rule:** Escalate only for **operational gaps**, never for emotional/relationship reasons.

**Valid triggers (only these):**
| Trigger | Signal phrase in stem |
|---|---|
| Explicit human request | "customer asks for a manager / human / person" |
| Policy gap or silence | "policy covers X but is **silent on** Y" / "no rule for this case" |
| Unable to make progress | all tools fail / blocked / no resolution path |
| Ambiguous identity | "multiple customer matches" / "need more identifiers" |

**Anti-triggers (look valid, are NOT):**
| Distractor | Why wrong |
|---|---|
| "Might damage relationship" / "delicate situation" | Sentiment — unreliable proxy |
| "Customer might change their mind" | Speculation, not a policy gap |
| "Contradictory evidence would be awkward" | Agent has the data; share it via standard dispute path |
| Multi-part request (billing + return) | Decompose & handle, don't escalate |
| Low self-reported confidence | Unreliable proxy |

**Disambiguator when two options look valid:** Is there a defined operational path (data + policy + tool)? → don't escalate. Is the agent missing a *rule*, not a *feeling*? → escalate.

---

## Pattern 5 — De-duplicating Noisy Output Across Runs

**Rule:** Eliminate duplicate/stale findings by giving Claude **prior state in context** and instructing it to report only new/unaddressed items. Don't filter mechanically; don't restrict scope.

**Reject:**
| Distractor | Why wrong |
|---|---|
| Post-processing string match on path + description | Brittle, loses real new issues that overlap, fragile to wording drift |
| Restrict review to modified files only | Misses cross-file regressions in unchanged files |
| Skip intermediate runs / only check initial + final | Defers problem; defeats iterative CI feedback |

**Generalization:** "Context-aware reasoning > coarse pre/post filters" applies to *any* "reduce noise without losing signal" question — also fits customer support history reuse, multi-agent shared findings, etc.

---

## Pattern 6 — Batch API Capabilities & Constraints

**Rule:** Batch API = **single-shot inference**. Tool *definitions* work; tool *call loops* don't. The async fire-and-forget model breaks iterative workflows regardless of latency.

**Batch API supports (don't get tricked):**
- Tool definitions in `tools` parameter ✅
- Claude emitting `tool_use` blocks ✅
- `custom_id` for request/response correlation ✅
- 50% cost discount ✅

**Batch API does NOT support:**
- Executing tools mid-request and feeding `tool_result` back ❌ (the iterative loop)
- Latency SLA — up to 24h ❌
- Real-time workflows ❌

**Suitable / Not suitable:**
| Workflow | Batch OK? |
|---|---|
| Iterative tool calling (multi-turn) | ❌ Architectural blocker |
| Blocking pre-merge CI / PR feedback | ❌ Latency |
| Real-time customer support | ❌ Latency |
| One-shot extraction on N independent docs | ✅ |
| Overnight reports / nightly batch jobs / weekly audits | ✅ |

**Trap to reject:** "Batch API doesn't support tool *definitions*" — false. It supports definitions; it doesn't support the *execution loop*. The primary technical constraint is the **async model**, not capability gaps.

**Exam keyword test:**
- "Iterative" / "continues analysis" / "tool calling loop" → architectural constraint (async model)
- "Real-time" / "blocking" / "pre-merge" → latency constraint
- "Overnight" / "weekly" / "independent items" → batch is suitable

---

## Pattern 7 — Reducing Tool-Call Round-Trips

**Rule:** Sequential tool calls that could be parallel → **prompt Claude to batch tool requests in one turn**. Native parallel tool calling is already supported.

**Reject:**
| Distractor | Why wrong |
|---|---|
| Speculative execution (call likely-needed tools alongside any request) | Wastes compute, pollutes context, may return misleading data, brittle heuristics |
| Composite tools (`get_customer_with_orders`) | Works but heavy: new code, combinatorial explosion, inflexible |
| Increase `max_tokens` | Misconception: `max_tokens` caps output length, not planning. Pure trap. |

**Hierarchy:** prompt-level batching → composite tools → speculative (almost never) → `max_tokens` (never).

**Why C wins over D:** native capability + prompt > new infrastructure when native works. C is strictly minimal change; D adds combinatorial maintenance.

---

## Pattern 8 — Multi-Agent Error Propagation

**Rule:** Subagent failures → return **structured error context** to the coordinator (failure type, attempted op, partial results, alternatives). Never generic statuses, silent empties, or workflow termination.

**Two-layer principle:**
1. Subagents handle **transient** failures locally (retry, scoped fallback)
2. Anything unresolved → propagate **structured context**, never opaque

**Reject:**
| Distractor | Why wrong |
|---|---|
| Local retry + generic "unavailable" status | Strips all info coordinator needs (failure type, query, partials) |
| Propagate to top-level → terminate workflow | One subagent failure kills the entire research; nuclear |
| Catch and return empty as success | LIES to coordinator → corrupts downstream synthesis |

**Required fields in propagated error context:**
- `failureType` (timeout / auth / rate limit / network / no results)
- `isRetryable` (true / false)
- `attemptedQuery` or operation
- `partialResults` (preserve, don't discard)
- `suggestedAlternatives`

**Trap watcher:** Options that include "*and after all retries return a generic status*" — the retry part is right, the return is wrong.

---

## Pattern 9 — Context Loading Mechanism (CLAUDE.md vs rules vs skills)

**Rule:** Match the loading trigger to **intent specificity**, not **file location**. When the same directory hosts multiple kinds of work and only one needs the context → **skill on-demand**.

**Loading mechanism by trigger:**
| Mechanism | Fires when | Use for |
|---|---|---|
| Project CLAUDE.md (root) | Every task in project | Truly universal context |
| Subdir CLAUDE.md | Any task touching subdir | Context relevant to all work in area |
| `.claude/rules/*.md` with path globs | Editing files matching glob | Conventions tied to file type/location, all work |
| **Skill (slash command / on-demand)** | **User explicitly invokes** | **Context useful only for specific task, not other work in same area** |
| Manual paste in prompt | Each request | One-off / experimental |

**Exam keyword decoder:**
| Question phrasing | Mechanism |
|---|---|
| "Always relevant" / "across all files" | CLAUDE.md (root) |
| "When working in directory X" / "files of type Y" | `.claude/rules/` with paths |
| "Only when doing specific task" / "**not for [other work in same area]**" | **Skill on-demand** |
| "Manually for one PR" | Inline paste |

**Trap signature:** Path-based rules feel surgical but fire for **all** work matching the path — including bug fixes, reviews, etc. If the constraint excludes other work in the same path, the answer is **skill**, not rules.

---

## Pattern 10 — Multi-Part Requests with Redundant Fetching

**Rule:** Sequential investigation + redundant data fetching across independent concerns → **decompose into concerns → shared context (fetch once) → investigate in parallel → synthesize**.

**Reject:**
| Distractor | Why wrong |
|---|---|
| Few-shot examples of multi-part sequences | Combinatorial input space; can't enumerate. Examples teach patterns, not structure |
| Consolidate tools into mega-tool (`investigate_issue`) | Hides reasoning, server-side speculative execution, loses flexibility |
| Verification gates between steps | **Enforces sequential** — opposite of what's needed |

**Composes with Pattern 7:** Pattern 7 batches *tool calls* within one investigation. Pattern 10 batches *concerns* within one request. Use both layered: parallel concerns, each emitting parallel tool calls.

**Keyword signature:** "investigating concerns sequentially" + "redundant data" + "complex multi-part" → decompose + shared context + parallel.

---

## Pattern 11 — Few-Shot Design (when few-shot IS the answer)

**Rule:** Few-shot examples must **target the failure mode** + **include visible reasoning** + **minimum effective dose**. Quantity ≠ quality.

**Failure-mode-targeted strategy:**
| Failure | Few-shot design |
|---|---|
| Ambiguous / edge cases | 4–6 ambiguous examples with reasoning ("why X over Y") |
| Format inconsistency | Format-focused examples |
| Tool ordering / parallelization | Examples showing the desired sequence |
| Diverse document shapes | Examples covering actual input variance |

**Reject these few-shot designs:**
| Distractor | Why wrong |
|---|---|
| 10–15 clear, unambiguous examples | Teaches what model already knows; wastes budget on easy cases |
| Examples grouped by tool/output | Hides comparative reasoning needed for ambiguous decisions |
| Generic best-practice examples | Doesn't target the actual failure surface |
| Many examples without reasoning | Pattern mimicry only; doesn't generalize to new shapes |

**Trap signature:** "More examples of clear cases" — wrong; budget belongs at the failure surface with reasoning visible.

**Distinct from Pattern 1 / 2:** Pattern 1 = vague criterion (not a few-shot problem). Pattern 2 = variable completeness (self-critique, not few-shot). This Pattern 11 = *when few-shot IS the right tool*, design it correctly.
