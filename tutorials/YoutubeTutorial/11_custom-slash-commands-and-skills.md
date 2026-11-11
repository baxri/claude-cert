# Understanding Custom Slash Commands and Skills in Claude Code

Exam-critical notes from the video transcript (Domain 3 — Claude Code Configuration & Workflows).

## The Scenario: "It Works for Me, Not for Sarah"

- A team uses a custom `/review` slash command in Claude Code
- A new teammate (Sarah) clones the repo, but `/review` doesn't exist for her
- **Root cause:** the command was created in the **user scope** (personal home directory), not the **project scope** — so it never traveled with the repo. Same class of bug as the `CLAUDE.md` scope issue from the previous lesson.

## Custom Slash Commands

- A slash command is like an Excel macro or IDE keyboard shortcut: instead of retyping a long prompt every time (e.g., "run all tests in this module and summarize failures"), you bundle the instructions into a markdown file and trigger it with `/command-name`
- Implementation: a `.md` file containing the full prompt/instructions — Claude reads and executes it when the command is invoked
- **More powerful than a simple text macro** — the file can contain a rich, detailed prompt, not just expanded text

### Where Commands Live (scope determines who can use them)

| Location | Scope | Shared with team? | Use case |
|---|---|---|---|
| `.claude/commands/` (in the project repo) | Project | Yes — version-controlled, everyone gets it on pull | Team-wide checks: security review checklist, PR templates |
| `~/.claude/commands/` (home directory) | User | No — personal to your machine only | Personal workflows, cross-project references (e.g., a command that navigates to another project on your machine to cross-reference, avoiding blowing your context by reading every project directly) |

**Exam keyword alert:** "available to all contributors/team members" → project scope (`.claude/commands/` in the repo).

**Sample exam pattern:** "Need a slash command enforcing a security review checklist, available to all team members" → store it in `.claude/commands/` in the project repo (not on each person's machine, not in `CLAUDE.md`).

### `CLAUDE.md` vs. Commands — Don't Confuse Them

- `CLAUDE.md` = onboarding/context documentation Claude reads automatically at the **start of every session** — describes the project, tech stack, conventions
- Commands = specific, on-demand instructions you invoke for a particular repeated task (e.g., code review, vulnerability check) — not loaded automatically, only run when called

## Skills

- A slash command is a **static prompt** — same instructions, same execution, every time
- A **skill** behaves more like a sub-agent: invoked on demand, has its own context, can be given specific tool access and constraints — much richer than a command
- Analogy: a command is like filling out a restaurant order form (kitchen makes exactly what's written); a skill is like hiring a specialist chef who brings their own tools and works independently toward an outcome you describe

### Where Skills Live

- `.claude/skills/<skill-name>/` — each skill gets its **own dedicated folder**
- The folder's most important file is `SKILL.md` (mandatory) — this defines the skill
- A skill's folder can also contain **optional supporting files**: templates (boilerplate code the skill references), a `reference.md` (important context the skill should keep in mind while executing) — these aren't mandatory, but add richness a plain command can't have

### Anatomy of `SKILL.md` (4 key front-matter fields)

1. **`name`** — the skill's name
2. **`context`** — `fork` or unset/default:
   - Unset → the skill runs **inside the current/main session**, consuming that session's context/tokens
   - `fork` → the skill spawns as a **separate sub-agent in its own session** (same underlying concept as fork sessions from the session-management lesson) — protects the main session's context window, and only the final result returns to the main session
3. **`allowed-tools`** — which built-in tools (e.g., Read, Grep) the skill is permitted to use — apply least privilege; don't grant tools the skill doesn't need
4. **`argument-hint`** — a prompt/hint shown to the user if they invoke the skill without providing a required argument (e.g., a file path)

### Skills vs. Commands — Practical Demo Takeaways

- A skill can be invoked **implicitly** by natural language (e.g., saying "create an agent" triggers a `create-agent` skill) — no explicit `/command` needed, if Claude recognizes the intent matches an existing skill
- A skill can **ask clarifying questions** before executing (e.g., "single-agent or multi-agent? mock data or real integration?") — a plain command just executes verbatim
- Without `context: fork`, a skill's work happens in the main session and consumes its tokens just like anything else — for a multi-step, token-heavy skill, forgetting `fork` risks exhausting the context window.

## `CLAUDE.md` vs. Skills vs. Commands (three-way comparison)

| | `CLAUDE.md` | Skill | Command |
|---|---|---|---|
| Loading | Passive — read automatically at session start | Loaded on demand | Activated on demand via `/command-name` |
| Isolation | No fork concept — runs in main session always | Can run isolated via `context: fork` | No isolation possible |
| Best for | Standing project rules, architectural guidance | Complex, multi-step procedures | Static, reusable prompts |

**Decision rule:** if you're tempted to put a 50-step procedural checklist into `CLAUDE.md`, that pollutes every session — move it to a **skill** instead (loaded/invoked only when needed, and forkable to protect context).

## Real-World Architecture Example: `/deep-review` Skill

1. Developer runs `/deep-review` (or a natural-language equivalent) on a target file
2. Claude reads `SKILL.md` and checks its `context` setting
3. If `context: fork` — spawns a sub-agent in its own session to do the actual deep review work (asking clarifying questions if needed, per the skill's defined steps)
4. The sub-agent finishes and returns **only the formatted findings** back to the main session — the same coordinator/sub-agent pattern from earlier lessons, applied natively inside Claude Code itself

## Sample Exam Question

**Q:** A team built a skill to analyze hundreds of files. They're worried intermediate results will flood their main conversation session. How do you fix this?
**A:** Set `context: fork`. (Distractors to reject: restricting `allowed-tools` to Read/Glob — that's about capability, not isolation; moving it to `.claude/commands/` in the home directory — commands can't fork, and this doesn't solve session pollution; `argument-hint` — solves a different problem entirely, missing arguments.)

## Cheat Sheet

- **Scope:** project scope (`.claude/commands/` or `.claude/skills/` in the repo) = shared with the team; user scope (`~/.claude/...`) = personal only
- **`context: fork` isolates *execution* (which session it runs in)** — it is not about restricting capability
- **`allowed-tools` restricts *capability* (which tools it can use)** — don't confuse the two; they solve different problems
- **Ambient vs. on-demand:** `CLAUDE.md` = ambient, loads every session automatically; skills/commands = on-demand, invoked only when needed
- **Complex, multi-step procedures → skills**, not a bloated `CLAUDE.md`

**Exam signal mapping:**
| Signal in the question | Answer |
|---|---|
| "Available to all contributors" | `.claude/commands/` (or `.claude/skills/`) in the project repo |
| "Prevent context pollution" / worried about flooding the main session | `context: fork` |
| "Principle of least privilege" for a skill's tool access | `allowed-tools` |
| "Requires a target file to run" / missing argument handling | `argument-hint` |
