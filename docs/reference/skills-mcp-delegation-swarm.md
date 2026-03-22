# Skills, MCP tools, delegation, and swarms (ZeroClaw)

This document describes how **Skills** and **MCP (Model Context Protocol) tools** fit into ZeroClaw, how to configure **sub-agents** (`delegate`) and **swarms**, and how work is handed off through prompts or tools. It reflects the current Rust implementation under `src/skills/`, `src/tools/mcp_*.rs`, `src/tools/delegate.rs`, and `src/tools/swarm.rs`.

---

## 1. Skills vs MCP “function calls”

### What a Skill is

In ZeroClaw, a **skill** is a **workspace-local capability package**: metadata plus instructions (and optional structured “tool” descriptions) loaded from disk.

- Default location: `<workspace>/skills/<skill-name>/` with either `SKILL.md` or `SKILL.toml` (see `src/skills/mod.rs`).
- Optional community mirror: **open-skills** can be synced when enabled in config (`[skills]`).
- At runtime, skills are merged into the **system prompt** as an `<available_skills>` XML block (names, descriptions, optional instructions and tool metadata). Injection mode is controlled by `skills.prompt_injection_mode` (`full` vs `compact`).
- **`read_skill`** is a built-in tool that loads the **full** skill source file on demand—mainly used in **compact** mode when instructions are not inlined in the system prompt (`src/tools/read_skill.rs`).

Skill entries can list **`SkillTool`** records (`name`, `description`, `kind`, `command`, …). These are **descriptive** entries surfaced to the model so it knows what the skill author intended (e.g. shell/http/script). They are **not** automatically registered as separate Rust `Tool` implementations. The model typically follows the skill by using normal ZeroClaw tools (`shell`, `file_read`, etc.) as appropriate.

### What an MCP tool call is

**MCP** connects ZeroClaw to **external MCP servers** over JSON-RPC (`tools/list`, `tools/call`). Each server is declared in `[mcp]` (`src/config/schema.rs`, `src/tools/mcp_client.rs`).

- Transports: **stdio** (local process), **HTTP**, or **SSE**.
- Each tool exposed by a server is registered under a **prefixed name**: `<mcp_server_name>__<tool_name>` to avoid collisions (`src/tools/mcp_client.rs`).
- Invocations go to the remote process/service; arguments and results are JSON. The wrapper `McpToolWrapper` implements the internal `Tool` trait and forwards calls after stripping any `approved` field used by ZeroClaw’s supervision model (`src/tools/mcp_tool.rs`).

### Key differences

| Aspect | Skills | MCP tools |
|--------|--------|-----------|
| **Nature** | Files + prompt text (and optional tool *metadata*) under your workspace | External processes or network services speaking MCP |
| **Execution** | Instructions in context; actions use **built-in** tools you allow | **Direct** RPC to the MCP server’s tool implementation |
| **Configuration** | `[skills]`, directories under `<workspace>/skills` | `[mcp]` servers (`enabled`, `servers`, `deferred_loading`, …) |
| **Discovery** | Listed in system prompt; `read_skill` for full file | Listed as tools (or stubs when deferred—see below) |
| **Naming** | Skill **name** as in `<available_skills>` | Prefixed MCP name: `myserver__some_tool` |

---

## 2. Adding Skills and MCP tools; triggering via prompts

### Adding a Skill

1. Create `<workspace>/skills/<name>/SKILL.md` (frontmatter + body) **or** `SKILL.toml` matching the schema in `src/skills/mod.rs`.
2. Optionally enable **open-skills** and paths via `[skills]` in `config.toml`.
3. Set `skills.allow_scripts` if you need script-like files (see security notes in config reference).
4. Choose **`prompt_injection_mode`**: `full` inlines more skill content; `compact` keeps context small and relies on `read_skill` when full instructions are needed.

#### Example layout: `SKILL.md` (minimal)

Place this at `<workspace>/skills/release-notes/SKILL.md` (folder name can match or differ from `name`; the parser sets `name` from the file):

```markdown
---
name: release-notes
description: "Draft concise release notes from a git diff or commit list."
version: "1.0.0"
tags: ["git", "docs"]
---

# Release notes skill

When asked to write release notes:

1. Use `shell` to run `git log` or `git diff` as needed (respect workspace sandboxing).
2. Summarize user-visible changes in sections: Added / Changed / Fixed / Security.
3. Keep bullets short; avoid internal refactors unless user-facing.

Optional tools (for the model’s planning only):

| name | kind | description |
|------|------|-------------|
| summarize | shell | Run git commands from workspace root |
```

#### Example `[skills]` block in `config.toml`

```toml
[skills]
# Community catalog (opt-in; see config reference for security audit behavior)
open_skills_enabled = false
# open_skills_dir = "/path/to/open-skills"

# full = inline instructions + tool metadata in system prompt (larger context)
# compact = names/descriptions/locations only; use read_skill for full text
prompt_injection_mode = "full"

# Allow .sh / shebang files in skill dirs (off by default)
allow_scripts = false
```

**Triggering by prompt:** There is no separate “skill command” the user must type. Once loaded, skills appear in the system prompt; the model can follow a skill when the user’s request matches the skill’s description. In compact mode, the model may call **`read_skill`** with `{ "name": "<skill-name>" }` to pull the full file.

#### Example user prompts (skills)

| Goal | Example user message |
|------|------------------------|
| Pick the skill implicitly | “Draft release notes for what we shipped since v1.2.” |
| Name the skill | “Follow the **release-notes** skill for this repo.” |
| Compact mode | “Use **read_skill** for **release-notes** if you need the full instructions, then summarize the last 10 commits.” |

#### Example `read_skill` tool call (model-facing JSON)

```json
{
  "name": "release-notes"
}
```

### Adding MCP servers / tools

1. Set `mcp.enabled = true` and add one or more `[[mcp.servers]]` entries (`name`, `transport`, `command` / `url`, `args`, `env`, `headers`, optional `tool_timeout_secs`).
2. **`mcp.deferred_loading`** (default `true`): only lightweight stubs plus the built-in **`tool_search`** tool are used initially; the model calls `tool_search` to load full schemas into an activated set (`src/tools/mcp_deferred.rs`, `src/tools/tool_search.rs`).
3. **`mcp.deferred_loading = false`**: every MCP tool is registered eagerly as a normal tool (higher context use).

#### Example `[mcp]` — stdio server (typical local MCP binary)

```toml
[mcp]
enabled = true
# Default true: stubs + tool_search; set false to register every MCP tool up front
deferred_loading = true

[[mcp.servers]]
name = "files"
transport = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/root"]
# tool_timeout_secs = 180
```

#### Example `[mcp]` — HTTP transport (sketch)

```toml
[[mcp.servers]]
name = "remote"
transport = "http"
url = "https://mcp.example.com/v1"
# headers = { Authorization = "Bearer ${TOKEN}" }  # use env/secret store in real setups
```

After a successful connection, tools appear in the registry as **`files__read_file`**, **`files__list_directory`**, etc. (exact names come from the server’s `tools/list` response).

**Triggering by prompt:** The primary agent (and any tool-capable path) chooses tools like any other capability: the model issues a tool call with the **prefixed** MCP name and JSON arguments. With deferred loading, it should use **`tool_search`** first to materialize the full tool definitions.

#### Deferred loading: typical flow

1. User: “Summarize `README.md` in the project root using the MCP filesystem server.”
2. Model calls **`tool_search`** with the names or patterns of tools it needs (per that tool’s schema).
3. Model calls the prefixed MCP tool, e.g. **`files__read_file`**, with arguments matching the loaded schema.

#### Example MCP tool call (eager or after activation)

Exact parameters depend on the MCP server. Illustrative shape:

```json
{
  "path": "README.md"
}
```

Tool name: **`files__read_file`** (not `read_file` alone).

#### Example user prompts (MCP)

| Situation | Example user message |
|-----------|------------------------|
| Explicit tool intent | “Use the **files** MCP server to list the workspace root, then read `Cargo.toml`.” |
| Deferred + search | “Search the MCP tool list for anything matching `*glob*` and then find Rust files under `src/`.” |

---

## 3. Sub-agents (`delegate`): provider, model, skills, MCP

### Configuration

Sub-agents are defined under **`[agents.<name>]`** in `config.toml` (`DelegateAgentConfig` in `src/config/schema.rs`). Important keys:

- **`provider`**, **`model`** (required): routed the same way as top-level providers.
- **`system_prompt`**, **`temperature`**, **`api_key`**, **`timeout_secs`**, **`agentic_timeout_secs`**, **`max_depth`**, **`max_iterations`**, **`allowed_tools`**, **`skills_directory`**.

Global defaults for delegate timeouts live under **`[delegate]`**.

#### Example `[delegate]` + `[agents.*]` in `config.toml`

```toml
[delegate]
timeout_secs = 120
agentic_timeout_secs = 300

# ── Non-agentic sub-agent: one model call, enriched prompt (skills/safety/workspace)
[agents.reviewer]
provider = "openrouter"
model = "anthropic/claude-sonnet-4"
system_prompt = "You are a concise code reviewer. Output only bullet findings."
temperature = 0.2
max_depth = 2
agentic = false
timeout_secs = 90

# ── Agentic sub-agent: tool loop with allowlist (use REAL tool names from ZeroClaw)
[agents.coder]
provider = "ollama"
model = "qwen2.5-coder:32b"
system_prompt = "You are an implementation assistant. Prefer small, safe edits."
temperature = 0.2
max_depth = 2
agentic = true
# Must match registered tools: e.g. file_read, file_edit, shell, web_search, ...
allowed_tools = ["file_read", "file_edit", "shell"]
max_iterations = 12
agentic_timeout_secs = 600

# Optional: only load skills from this folder relative to workspace (not merged with default skills/)
# skills_directory = "skills/coder-only"
```

**Note:** Tool names must match what ZeroClaw registers (for example **`file_read`**, **`file_edit`**, **`shell`**). Older examples sometimes used aliases like `read` / `edit` / `exec`; those will **not** match unless your build registers tools under those exact strings.

### Two modes

1. **`agentic = false` (default)**  
   One **user message** to the sub-agent’s provider: `chat_with_system` with an **enriched** system prompt (tools section empty here, but skills/workspace/safety sections are included—see `build_enriched_system_prompt` in `src/tools/delegate.rs`).

2. **`agentic = true`**  
   Runs the shared **`run_tool_call_loop`** with a **filtered** tool registry: only tools whose names appear in **`allowed_tools`**, and **`delegate` is always excluded** to prevent re-entrant delegation loops. Iterations are capped by **`max_iterations`**.

### Skills for sub-agents

- If **`skills_directory`** is set (relative to workspace), skills are loaded **only** from that directory; otherwise the default `<workspace>/skills` is used (`delegate.rs`).
- Skills are injected with **`SkillsPromptInjectionMode::Full`** for the sub-agent’s enriched prompt.

### MCP for sub-agents

Agentic sub-agents receive tools from the **`parent_tools`** list: a clone of the primary registry **plus** any tools pushed after construction (e.g. eagerly registered MCP wrappers).

- With **`mcp.deferred_loading = false`**, you can list MCP tools in **`allowed_tools`** using their **prefixed** names (`server__tool`).
- With **`mcp.deferred_loading = true`**, the primary agent uses **`tool_search`**; that path does **not** add MCP wrappers to the delegate parent-tool list the same way as eager registration. For agentic sub-agents that must call MCP tools directly, **eager registration** is the straightforward option.

Example agentic allowlist including an MCP tool (eager MCP only):

```toml
[agents.mcp_helper]
provider = "openrouter"
model = "anthropic/claude-sonnet-4"
agentic = true
allowed_tools = ["file_read", "files__read_file"]
max_iterations = 8
```

### Triggering sub-agents

The primary agent (or the user) does **not** use a special string—sub-agents are invoked when the model calls the **`delegate`** tool:

```json
{
  "agent": "<configured-agent-name>",
  "prompt": "<task>",
  "context": "<optional context>"
}
```

**Recursion** is limited by **`max_depth`** on the target agent’s config and by excluding **`delegate`** from agentic allowlists.

#### Example user prompts (delegation)

| Goal | Example user message |
|------|------------------------|
| Explicit agent + task | “Use **delegate** to send this to agent **reviewer**: prompt = ‘Review `src/main.rs` for panic risks’, context = paste the file if needed.” |
| Natural hand-off | “Have the **coder** sub-agent implement a `fmt` helper in `src/lib.rs` and run tests.” |
| Context separation | “Delegate to **reviewer** with prompt ‘Are we missing error handling?’ and put the stack trace in **context**.” |

#### Example `delegate` tool call (model-facing JSON)

```json
{
  "agent": "coder",
  "prompt": "Add a unit test for `parse_config` in `src/config/mod.rs` and run `cargo test` for that module.",
  "context": "Workspace is a Rust crate; keep changes minimal."
}
```

---

## 4. Agent swarms

### What a swarm is

A **swarm** is a **named orchestration** over a list of **`[agents.*]`** entries. It is implemented as the **`swarm`** tool (`src/tools/swarm.rs`). Strategies (`SwarmStrategy`):

- **`sequential`**: agents run one after another; each step’s **text output** becomes the next step’s working input (with the original task preserved in the prompt).
- **`parallel`**: every agent runs the **same** full prompt (plus optional context) **concurrently**; outputs are concatenated.
- **`router`**: an LLM call using the **first** swarm member’s provider chooses **one** agent name; that agent then runs the task.

Configuration lives in **`[swarms.<name>]`**: `agents`, `strategy`, optional `router_prompt`, `description`, `timeout_secs`.

#### Example `[agents.*]` + `[swarms.*]` in `config.toml`

Swarms only reference **names** of agents defined in `[agents.<name>]`. Strategies are `sequential`, `parallel`, or `router`.

```toml
[agents.researcher]
provider = "openrouter"
model = "anthropic/claude-sonnet-4"
system_prompt = "You are a researcher. Produce factual bullets with short citations or search queries."
temperature = 0.3

[agents.writer]
provider = "openrouter"
model = "anthropic/claude-sonnet-4"
system_prompt = "You are a technical writer. Turn rough notes into polished prose."
temperature = 0.5

[agents.fast]
provider = "ollama"
model = "llama3.2"
system_prompt = "You give very short answers."

# Pipeline: researcher output → writer input (see Data transfer below)
[swarms.blog_pipeline]
agents = ["researcher", "writer"]
strategy = "sequential"
timeout_secs = 600
description = "Research then write-up for blog drafts"

# Same prompt to every agent at once; outputs concatenated
[swarms.multi_review]
agents = ["researcher", "fast"]
strategy = "parallel"
timeout_secs = 300

# Router picks ONE agent using an extra LLM call (first agent's provider is used for routing)
[swarms.auto_pick]
agents = ["researcher", "writer", "fast"]
strategy = "router"
router_prompt = "Pick the single best agent for the user's task based on role descriptions."
timeout_secs = 300
```

### Creating a workflow

1. Define **`[agents.*]`** entries for each participant.
2. Add **`[swarms.<name>]`** referencing those agent names and a strategy.
3. Ensure the **`swarm`** tool is available (it is registered when at least one swarm exists—`src/tools/mod.rs`).

### Triggering

- **Via prompt / tool use:** The model calls **`swarm`** with:

```json
{
  "swarm": "<configured-swarm-name>",
  "prompt": "<task>",
  "context": "<optional>"
}
```

- **Other methods:** Any entrypoint that runs the main agent loop with tools (CLI `zeroclaw agent -m`, WebSocket/gateway, channels) can eventually lead to a **`swarm`** tool call the same way—there is no separate HTTP “swarm-only” endpoint in the core tool layer; orchestration is **tool-driven**.

#### Example user prompts (swarm)

| Strategy | Example user message |
|----------|------------------------|
| Sequential | “Run **swarm** **blog_pipeline** with prompt ‘Outline pros/cons of SQLite for embedded IoT’ and no extra context.” |
| Parallel | “Use **swarm** **multi_review**: prompt ‘Is this API design sane?’ and context = paste the OpenAPI fragment.” |
| Router | “Call **swarm** **auto_pick** with prompt ‘Translate this paragraph to Spanish’—let the router choose the agent.” |

#### Example `swarm` tool call (model-facing JSON)

```json
{
  "swarm": "blog_pipeline",
  "prompt": "Draft sections for a post on Rust async runtimes; audience is senior backend engineers.",
  "context": "Our product already uses Tokio 1.x."
}
```

#### Sequential strategy: what the user’s text becomes (conceptual)

For agent index `i > 0`, the implementation builds a prompt like:

```text
[Previous agent output]
<full text returned by agent i-1>

[Original task]
<the same `prompt` string passed to the swarm tool>
```

So the **data passed between sub-agents** is only the **prior agent’s plain text output** plus the **original task**—not tool transcripts unless the model included them in that text.

### Data transfer between agents

- **Sequential:** The previous agent’s **plain text output** is fed into the next prompt; the **original user task** string is repeated in the prompt for context (`execute_sequential` in `swarm.rs`).
- **Parallel:** No shared handoff—each agent sees the same **full_prompt** (context + task).
- **Router:** Only the **chosen** agent runs; prior agents do not produce intermediate data.

**Note:** Swarm **`call_agent`** uses a single **`provider.chat_with_system`** with `agent_config.system_prompt` only. It does **not** run the **agentic** tool loop and does **not** apply the richer **delegate** system prompt (skills sections, tool lists, etc.). So **`agentic` / `allowed_tools` / `skills_directory` on `[agents.*]` do not affect swarm members** the way they do for **`delegate`**.

### Loops and exit conditions

**Swarm:** Each strategy is **finite** (fixed list of agents or one router call + one execution). Time limits use **`timeout_secs`** (and per-step splits for sequential). There is no arbitrary “loop” inside a swarm beyond that.

**Agentic `delegate` and the main agent loop** use `run_tool_call_loop` (`src/agent/loop_.rs`). The loop stops when:

- The model returns **no tool calls** (final natural-language answer),
- **`max_iterations`** (or the configured max tool iterations) is reached,
- **Cancellation** fires,
- Other **guardrails** apply (e.g. **budget** checks, **model switch** handling, optional **identical-output** detection when pacing is configured—see comments around `run_tool_call_loop`).

For nested delegation, **`max_depth`** prevents infinite delegation chains.

---

## Quick reference: what to put in `config.toml`

| Topic | Section | Remember |
|--------|---------|----------|
| Skill load behavior | `[skills]` | `prompt_injection_mode`, optional `open_skills_*`, `allow_scripts` |
| MCP servers | `[mcp]` + `[[mcp.servers]]` | `enabled`, `deferred_loading`, per-server `name` → **`name__tool`** |
| Delegate timeouts | `[delegate]` | Defaults; overridable per agent |
| Sub-agents | `[agents.<id>]` | `provider`, `model`, `agentic`, `allowed_tools`, `skills_directory`, … |
| Swarms | `[swarms.<id>]` | `agents = [...]`, `strategy`, `timeout_secs`, optional `router_prompt` |

---

## Example: one-shot CLI message (primary agent)

The primary agent decides whether to use normal tools, **`delegate`**, **`swarm`**, **`read_skill`**, or MCP tools. You do not configure that in TOML— you describe the goal in natural language:

```bash
zeroclaw agent -m "Research ZeroClaw delegate vs swarm in the docs, then tell me when to use each."
```

If **`delegate`** / **`swarm`** are registered and the model is allowed to call them, it may issue those tool calls autonomously. Exact behavior depends on provider, system prompt, and tool policy.

---

## See also

- `docs/reference/api/config-reference.md` — full TOML tables for `[skills]`, `[mcp]`, `[agents.*]`, `[delegate]`, and swarm examples in tests (`src/config/schema.rs` around swarm deserialization).
- `examples/config.example.toml` — minimal `[delegate]` / `[agents.*]` sample (verify tool names against your build).
- `CLAUDE.md` (repository root) — validation commands and architecture map.
