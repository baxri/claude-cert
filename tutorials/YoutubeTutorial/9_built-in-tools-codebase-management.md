# Mastering Claude Code's Built-In Tools for Efficient Codebase Management

Exam-critical notes from the video transcript (Domain 3/4 — Claude Code Configuration & Workflows, maps to syllabus task 2.5). Covers the six core built-in tools: Grep, Glob, Read, Write, Edit, Bash.

## Why This Matters

No MCP setup, API keys, or config needed — these are built-in and always available. The exam (and real interviews) test whether you know **which tool to use for which situation**; using the wrong one wastes tokens and produces unreliable results. Tested in at least two exam scenarios.

## Grep — the Content Detective

- Searches **inside** file contents for a pattern; does not care about file names
- Use when you need to find *where a specific string/function/pattern appears* (e.g., "find every call to `process_refund`")
- **Common misconception to avoid:** don't invoke grep via the Bash tool — there's a dedicated Grep tool for a reason (straightforward, purpose-built); reaching for `bash` + `grep` when the Grep tool exists is flagged as incorrect reasoning on the exam

## Glob — the File Path Hunter

- Matches file **names/paths** by pattern; never looks inside file contents
- Analogy: grep opens each folder and reads the documents inside; glob only reads the labels on the folders — never opens them
- Use for: "give me all `.py` files in `src/`," "find every test file," "list all markdown docs"

### Grep vs. Glob — Classic Mix-Up (exam pattern)

| Task | Tool | Why |
|---|---|---|
| Find every file that imports React | Grep | `import` is file content |
| Find all TypeScript files in `/src` | Glob | File extension = naming pattern |
| Find every test file in the project | Glob | `.test.*` naming pattern |
| Find all calls to `process_refund` | Grep | Function usage is inside file content |
| List all files in a Terraform directory | Glob | Discovering files by path structure |
| Find all TODO comments | Grep | Comments live inside file content |

## Read — Load It All In

- Loads the **full contents of one specific file** into context
- Use when you already know which file you need (e.g., grep pointed you to `refunds.py` line 42, now you read the whole file to understand it)
- **Trade-off:** expensive at scale — reading many files up front (e.g., 20-30 files "just to explore") can burn through most of the context window before any real reasoning happens

## Write — Create and Overwrite

- Creates a new file, or **completely replaces** an existing file's contents — an atomic, destructive operation
- Use when: (1) creating a brand-new file that doesn't exist yet (the only option), or (2) changes are so extensive that editing piecemeal is impractical and full regeneration is cleaner
- Also serves as the **fallback mechanism when Edit fails** (see below)

## Edit — Targeted, Surgical Changes

- Makes a targeted modification to a specific section of a file (find-and-replace) without rewriting the whole thing
- Efficient and low-risk for unrelated code — ideal for a 500-line file where only 3 lines need to change
- **Critical constraint (high-yield exam fact): the `old_string` being replaced must be 100% unique within the file.** If that exact text appears more than once, Edit doesn't know which occurrence to target and fails.

### The Edit Failure Fallback Pattern (canonical, exam-tested)

When Edit fails because the anchor text isn't unique, the correct recovery is:
1. **Read** — load the full file into context
2. **Reason** — the agent modifies the code in memory
3. **Write** — overwrite the file with the updated content

This read → reason → write fallback is less elegant than a targeted edit (rewrites the whole file instead of a fragment) but always works, since it has no uniqueness requirement.

**Pro tip:** to make Edit more likely to succeed on the first try, use a **longer, more specific `old_string`** (e.g., the full unique declaration line, not just a short fragment likely to repeat elsewhere).

## Bash — the Escape Hatch

- Executes shell commands: run tests, install packages, call CLIs, git operations (status, etc.)
- Use for anything that's a genuine shell/terminal operation — not as a substitute for Grep/Glob/Read/Write/Edit when a dedicated tool exists for the task

## The Incremental Exploration Pattern (Grep → Read → Trace)

- **Anti-pattern:** a naive agent given "explore this codebase and understand the billing system" reads every file in the relevant directory all at once — burning the context window before any reasoning happens
- **Correct pattern:** incremental exploration —
  1. **Grep** for the entry point / key function (e.g., `def process_refund`)
  2. **Read** the specific file grep identified
  3. **Trace** the call chain to find the next dependency
  4. **Grep** again for that dependency, **read** again, repeat
- Each grep call informs the next read; each read reveals the next thing to grep for — building understanding incrementally while keeping the context window efficient
- **Exam signal:** if an answer choice says "read all files first" (bulk reading), it is almost certainly wrong.

## Full Workflow Example — Developer Productivity Scenario

1. **Discovery (Glob):** map the folder structure — e.g., find all Python files in the `orders` module
2. **Understanding (Grep → Read → trace):** grep for the main entry point (e.g., `class OrderProcessor`) in the discovered path → read that file → trace referenced constants/status codes → grep/read those too
3. **Create (Write):** generate new boilerplate (e.g., a new `order_status.py` endpoint) based on what was learned
4. **Fix (Edit):** surgically patch a specific legacy issue found during exploration (e.g., replacing an old status code constant), using a unique anchor string

This discovery → understanding → create/fix flow generalizes to how Claude Code and similar coding assistants operate on real tasks.

## Cheat Sheet

| Tool | Role | Key rule |
|---|---|---|
| **Grep** | Search file **contents** | Not for finding files by name |
| **Glob** | Match file **paths/names** | Doesn't look inside files |
| **Read** | Load one specific file fully | Expensive at scale — use targeted, not bulk |
| **Write** | Create or fully overwrite a file | Destructive; also the Edit-failure fallback |
| **Edit** | Targeted find-and-replace in a file | `old_string` must be unique in the file |
| **Bash** | Run shell commands | Tests, installs, CLIs, git — not a grep substitute |

**Three facts to lock in for the exam (task 2.5):**
1. Grep = contents, Glob = paths
2. Edit requires a unique anchor string; on failure, fall back to Read + Write
3. Incremental exploration (grep → read → trace → repeat) always beats bulk reading
