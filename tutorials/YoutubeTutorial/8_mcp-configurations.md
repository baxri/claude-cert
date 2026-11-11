# Understanding MCP Configurations: How Agents Discover and Use External Tools

Exam-critical notes from the video transcript (Domain 2 — Tool Design & MCP Integration).

## What MCP Is and Why It Exists

- Before MCP, tools were hardcoded/embedded directly inside each agent's code — tightly coupled, not reusable across agents, brittle to change
- **MCP (Model Context Protocol)**, developed by Anthropic, is an **open protocol** that standardizes how AI applications/agents discover and use external capabilities: tools, resources, and prompts
- Analogy: MCP is like "USB-C for AI" — one standard interface any agent or coding assistant can plug into
- Under the hood, MCP is **JSON-RPC over HTTP** (or STDIO for local servers) — not a fundamentally new transport, just a standardized contract

## MCP Architecture

- **Host** = the agent itself (contains an MCP client/SDK)
- **MCP server** = where tools, resources, and prompts actually reside (can be local or remote)
- MCP is **stateful, not stateless** — a session persists across a conversation so the agent doesn't lose context of prior tool calls

## Communication Flow (5 phases — exam-relevant)

1. **Connect** — host connects via STDIO (local) or HTTP/streamable HTTP (remote)
2. **Initialize** — client and server establish a session and negotiate capabilities (session ID for stateful tracking)
3. **Discover** — agent calls `tools/list` (and similar methods) to dynamically learn what tools/resources/prompts the server offers, instead of hardcoding tool knowledge
4. **Reason** — the LLM now has context: what tools exist, what they do, what arguments they expect
5. **Invoke** — the agent calls a specific tool when needed, and the server returns structured results back through the loop

## MCP vs. Hardcoded Tools vs. APIs

- MCP advantages over hardcoded tools: runtime tool discovery, reuse across multiple agents/apps, metadata-driven understanding (vs. brittle code coupling)
- **MCP does not replace APIs.** APIs remain how services/microservices communicate with each other; MCP is specifically about **AI-facing tool/resource discovery**. An API can simply be wrapped behind an MCP server to make it agent-discoverable without changing the underlying service.
- Practical benefit: if a wrapped API's contract changes, you update the MCP server, not every agent that consumes it.

## Real-World Use Cases

- Developer assistants/IDEs (Claude Code, coding assistants) connecting to internal/private company MCP servers (behind auth) to access internal docs/tools not in public training data
- Enterprise knowledge assistants, design-to-code workflows, incident ops/response
- Public MCP servers exist (OpenAI docs, GitHub, filesystem, memory, fetch, etc.) that can be tested directly

## Security Considerations

- MCP does not replace your existing security model — keep using OAuth, secrets/access keys, proper authentication regardless of whether something is exposed via MCP
- Watch for prompt injection / tool poisoning risks if communication isn't properly secured
- Enforce least-privilege permissions on what an MCP server exposes

## Testing MCP Servers

- Public/well-known MCP servers (OpenAI, GitHub) can often be tested directly via `curl`/Postman with raw JSON-RPC calls (`initialize`, `tools/list`, `tools/call`)
- For authenticated/corporate MCP servers, use **MCP Inspector** (a tool built by Anthropic) rather than manually managing sessions — it handles initialize/list/call for you
- Coding assistants (Claude Code, Cline/Klein, Codex, etc.) have built-in settings to register remote MCP servers so the assistant can pull in organization-specific context during coding tasks

---

## MCP Configuration Files (core exam topic)

### The Two Scopes

| File | Location | Scope | Version-controlled? |
|---|---|---|---|
| `.mcp.json` | Project root | Things the **project** needs (which MCP servers/tools the agent uses) | **Yes** — shared with the team |
| `~/.claude.json` (`.claude.json` in home dir) | User home directory | Things **you personally** need (personal API keys, SSH credentials, preferences) | **No** — never shared |

**Exam trap (classic scenario):** "Agent worked perfectly locally but teammates who pulled the repo have no MCP tools." Root cause: the MCP server configuration was placed in the personal `~/.claude.json` instead of the project-level `.mcp.json`. **Fix:** move the config into `.mcp.json` at the project root.

### `.mcp.json` Structure

- Top-level `mcpServers` object; each key names a server and defines how Claude connects to it
- Two connection flavors:
  - **STDIO servers** — local; Claude spawns them as a subprocess and communicates over standard input/output (e.g., `command: npx`, with `args`)
  - **Remote servers (HTTP/SSE)** — Claude acts as an HTTP client connecting to a network URL; best for hosted APIs
- **The choice of transport is driven by where the server lives, not what it does** — the same functionality could be exposed either way.

### Environment Variables (secrets)

- Never hardcode secrets (API tokens, DB passwords, webhook URLs) into `.mcp.json` since it's committed to version control
- Use **environment variable expansion syntax** — e.g., `${DB_PASSWORD}` — Claude resolves the value from the shell environment at runtime (however that environment is populated: exported locally, set in a container, etc.)
- **Exam trap:** reject any answer using `$VAR_NAME` or `%VAR_NAME%`-style syntax without the `${...}` braces — the correct MCP syntax is `${VAR_NAME}`. Values stay local (environment); variable names ship in git (`.mcp.json`).

## MCP Resources — the "Context Shortcut"

- **Problem without resources:** if an MCP server only exposes tools (e.g., a docs server with 500 articles), Claude must burn multiple exploratory tool calls (search, re-search with different terms, read, search again) before finding relevant content — expensive in tokens and latency
- **Fix — MCP resources:** the server can expose static/structured content directly as a catalog that shows up in Claude's context up front, so Claude navigates directly instead of repeatedly searching
- **Exam framing:** the architectural shift is stopping Claude from burning multiple exploratory tool calls when exposing a resource catalog achieves the same result in effectively one step
- Whether resources are available depends on how the MCP server author built it — well-designed public servers (GitHub, OpenAI docs) tend to expose them; for a custom server, you design and expose resources yourself (e.g., via a framework like FastMCP)

## Community vs. Custom MCP Servers

- **Community servers** — publicly available, ready to use (GitHub, OpenAI docs, filesystem, etc.)
- **Custom servers** — built and hosted internally when you need to expose organization-specific/internal data with your own authentication
- **Recommended pattern:** start with a community server, then enhance its generic tool descriptions via **overrides** in your `.mcp.json` rather than building a custom server from scratch when not necessary

## Tool Description Competition with Built-ins

- Claude has **built-in tools** (Grep, Glob, Bash, etc.) that it knows natively/intimately
- When you add an MCP tool that does something similar (e.g., a semantic code search tool vs. built-in grep), your MCP tool's description must **compete** for Claude's selection
- A vague MCP tool description → Claude defaults to the familiar built-in tool
- A rich, specific description (what it does, when to use it *instead of* the alternative) → Claude prefers the MCP tool when appropriate
- **Exam rule:** enhancing the tool description is always the **first, highest-leverage fix** for incorrect tool selection — same principle covered in earlier lessons, now applied specifically to MCP tools competing with built-ins.

## No Lazy Loading of Tools

- Claude loads **all** configured tools into context at once — there is no on-demand/lazy loading of tool definitions
- Advantage: low latency, since everything needed is already in context
- Risk: **context pollution** — too many tools (especially with weak descriptions) degrades tool-selection reliability
- Mitigations (consistent with earlier lessons): scope tools per sub-agent/role, use hooks to control execution, don't expose one agent to another agent's tools

## Decision Framework (cheat sheet)

| Situation | Action |
|---|---|
| Tool/config needs to be shared with the team | Put it in `.mcp.json` at project root, not `~/.claude.json` |
| Secrets (tokens, passwords) | Use `${ENV_VAR}` expansion — never hardcode |
| Static content/docs the agent needs often | Expose via MCP **resources**, not repeated tool calls |
| A need already covered by a public server | Use a community server; add tool-description **overrides** if needed |
| No suitable public server / internal-only data | Build a **custom** MCP server with proper auth |
| Agent keeps defaulting to built-in tools (e.g., grep) instead of your MCP tool | Improve/enhance the MCP tool's description first |
| Teammate reports missing MCP tools after pulling the repo | Move config from `~/.claude.json` to project-root `.mcp.json` |
