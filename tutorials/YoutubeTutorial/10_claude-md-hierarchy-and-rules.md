# Mastering Claude Code Configurations: Resolving the Missing Instructions Problem

Exam-critical notes from the video transcript (Domain 3 — Claude Code Configuration & Workflows).

## The Scenario: "Works on My Machine"

- A developer spends weeks getting Claude Code dialed into their project's conventions (style, tech stack rules, etc.)
- A new teammate clones the same repo, runs their own Claude Code session — and Claude acts like it's never seen the project, because the original developer's configuration wasn't committed to the repo
- This lesson is about understanding `CLAUDE.md` (and related mechanisms) well enough to avoid that failure mode.

## What `CLAUDE.md` Is

- **Persistent context** — content Claude reads *before* the conversation even starts (loaded when you trust the folder / start a session)
- Functions like a briefing document you'd hand a new developer: tech stack, conventions, architectural notes — so you don't have to re-explain it (and burn tokens having Claude re-discover the whole project) every session

## The Three-Level Hierarchy (most-specific wins concept, but all three can combine)

### 1. User Level
- Location: `~/.claude/CLAUDE.md` (home directory)
- Scope: applies to **all of your own projects**, on your machine only
- **Never shared** — not part of any repo, doesn't propagate to teammates
- Good for: personal preferences/style (e.g., "I prefer verbose commit messages that explain why")

### 2. Project Level
- Location: either the **project root** (`CLAUDE.md`) or inside a dedicated **`.claude/` folder** (`.claude/CLAUDE.md`) — both are valid; team preference decides (root = more discoverable, `.claude/` = keeps things organized)
- Scope: **committed to the repo** — the whole team + CI get the same file on pull/clone
- Good for: shared team knowledge — setup instructions, conventions, tool/error-handling patterns, cost/model-choice guidance, etc.

### 3. Directory Level
- Location: a `CLAUDE.md` inside any subdirectory (e.g., `infrastructure/CLAUDE.md`, `packages/api/CLAUDE.md`, `packages/frontend/CLAUDE.md`)
- Scope: loads **only when Claude Code is working in that directory** — the most surgical/specialized level
- Good for: directory-specific guidance (Terraform conventions in `infrastructure/`, NestJS patterns in the API package, React rules in the frontend package)
- The mechanism is identical to the other levels — just scoped more narrowly.

## The Classic Team Bug (exam-tested scenario)

- **Root cause:** the original developer put their `CLAUDE.md` in their **user-level home directory** instead of the project. It worked great for them, but was never committed.
- **Symptom:** a new engineer clones the repo, runs Claude Code, and gets none of that context — because user-level config never leaves that machine.
- **Fix:** move the file to the project root (`CLAUDE.md`) or `.claude/CLAUDE.md`, and commit it — every teammate gets the same context on pull.
- **Exam alert:** "teammate isn't following project conventions / Claude doesn't know project context" → check the config's **location** first (user-level vs. project-level).

## `@import` — Keeping `CLAUDE.md` Modular

- As a project grows, a single `CLAUDE.md` can balloon into a "5,000-line monster" covering architecture, testing, API patterns, deployment rules, glossaries, etc.
- **Fix:** `@import` lets you split configuration into focused files and compose them — e.g., a separate `api-patterns.md`, a separate `testing-conventions.md`, referenced from the main `CLAUDE.md`
- Benefit: teams only touch the file relevant to their concern (backend team edits `api-patterns.md`, testing team edits `testing-conventions.md`) — fewer merge conflicts, cleaner PRs, clearer ownership
- Conceptually the same idea as modularizing code into imported utility/service files.

## `.claude/rules/` Directory — Conditional, Topic-Specific Rules

- A **separate mechanism** from `CLAUDE.md` files: topic-specific rule files, each focused on one area of the codebase/workflow
- Uses **YAML front matter** with two key fields:
  - `description` — always present, explains what the rule covers (e.g., "TypeScript conventions: never use `any`, use `unknown` with type narrowing")
  - `paths` — makes the rule **conditional**: it only applies to files matching the given glob pattern(s)
- **Key distinction from `CLAUDE.md`:** `CLAUDE.md` (at any of the 3 levels) cannot be scoped to a *file pattern* — it's scoped to a *location* (whole project, or a whole directory). `.claude/rules/` + `paths` lets a rule apply based on file type/pattern **regardless of which directory the file lives in**.
- **Exam signal:** ".claude/rules/ = topic-specific rule files with front matter; `description` is always present; adding `paths` makes the rule conditional on file patterns."

## Glob Patterns in `paths` (file-targeting)

Same glob syntax used elsewhere in Claude Code (e.g., `.gitignore`-style):

| Pattern | Meaning |
|---|---|
| `terraform/**/*` | All files under `terraform/` at any depth |
| `**/*.test.tsx` | Any `.test.tsx` file anywhere in the project |
| `**/*.spec.ts` | Any `.spec.ts` file anywhere in the project |
| `src/api/*.ts` | Only `.ts` files directly inside `src/api/` (not nested further) |
| `**/*.{ts,tsx}` | Both `.ts` and `.tsx` files anywhere in the project |

- `**` = any number of directories (any depth)
- `*` = anything within one directory level / any filename matching the rest of the pattern
- Patterns apply regardless of directory depth when `**` is used.

## `/memory` Command — Debugging What's Loaded

- Use `/memory` to see exactly what `CLAUDE.md` files and rules Claude has loaded into the current session (project memory, user memory, etc.)
- **Exam use case:** if a question asks "how do you verify/debug whether Claude is loading your configuration files," the answer is the **`/memory` command** — a diagnostic tool, not a fix by itself.

## Sample Exam Pattern

**Q:** Need different rules for TypeScript files (`src/**/*.ts`) and Terraform files (`terraform/**/*`) — how do you apply each only to its own file type?
**A:** `.claude/rules/` with YAML `paths` front matter — purpose-built for glob-based conditional loading. (Not `@import`, not directory-level `CLAUDE.md` with path prefixes — `CLAUDE.md` isn't pattern-conditional.)

## Cheat Sheet

| Level/Mechanism | Location | Who sees it | Purpose |
|---|---|---|---|
| User-level `CLAUDE.md` | `~/.claude/CLAUDE.md` | Only you | Personal preferences across all your projects |
| Project-level `CLAUDE.md` | repo root or `.claude/CLAUDE.md` | Whole team + CI (committed) | Shared team conventions |
| Directory-level `CLAUDE.md` | any subdirectory | That directory (and below) | Directory-specific guidance |
| `.claude/rules/` + `paths` | `.claude/rules/*.md` | Whole team (committed) | Conditional, glob-scoped rules regardless of directory |
| `@import` | referenced from `CLAUDE.md` | Whole team | Modularizes a growing `CLAUDE.md` into focused files |
| `/memory` | command | N/A | Diagnostic — shows what's actually loaded in the session |

**Three facts to lock in for the exam:**
1. User-level `CLAUDE.md` is **never shared** — it's the classic root cause of "works for me, not for my teammate."
2. `.claude/rules/` + `paths` is the answer whenever a question needs conditional application **by file type/pattern**, not by directory.
3. `/memory` is the diagnostic command for config debugging — not a fix, just visibility into what's loaded.
