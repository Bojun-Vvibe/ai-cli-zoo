# claude-agent-sdk-python

> Snapshot date: 2026-04. Upstream:
> <https://github.com/anthropics/claude-agent-sdk-python>
> Last verified version: **v0.1.68** (released 2026-04-25). License file:
> [`LICENSE`](https://github.com/anthropics/claude-agent-sdk-python/blob/main/LICENSE)
> (blob sha `3fa6a64e52f30d3ad836f98b3f0da6f4b6263bb8`) — MIT.

Anthropic's first-party Python SDK for **building** agent CLIs against
the same harness that powers `claude-code`. It is not itself a coding
CLI you install and run on a repo — it's the toolkit you reach for when
you want to ship your *own* in-terminal agent that has Claude's
file-edit / shell-exec / MCP / sub-agent / hook semantics without
re-implementing the loop. Functionally: the public surface that the
`claude-code` binary uses internally, packaged for third-party Python.

It belongs in this catalog because almost every "I need a custom Claude
agent for X" greenfield CLI built in 2026 starts here instead of
hand-rolling a tool loop on top of the raw `anthropic` SDK.

## 1. Install footprint

- `pip install claude-agent-sdk` (or `uv add claude-agent-sdk`).
- Pure Python wheel + a Node.js dependency: the SDK shells out to the
  `@anthropic-ai/claude-code` Node package for the actual sandboxed
  tool execution layer. So `node >= 18` and `npm i -g
  @anthropic-ai/claude-code` are de-facto requirements for the
  file-edit / shell-exec tools to work.
- Python 3.10+. macOS, Linux, Windows (WSL strongly recommended for the
  Node sandbox path).
- Configuration: `ANTHROPIC_API_KEY` env var, or pass `api_key=` to the
  client. MCP servers configured via `~/.claude.json` (same file the
  Node CLI reads) or programmatically per session.

## 2. License

MIT. License file at the repo root: `LICENSE` (blob sha
`3fa6a64e52f30d3ad836f98b3f0da6f4b6263bb8` at tag `v0.1.68`). Note that
the bundled `@anthropic-ai/claude-code` Node package it shells out to
is *source-available* under Anthropic's commercial license — your
greenfield CLI's license story is MIT-on-Python + commercial-on-Node-runtime.

## 3. Models supported

Claude only — Opus, Sonnet, Haiku across the 3.5 / 3.7 / 4.x lines via
the standard Anthropic API. No multi-provider abstraction (this is the
explicit non-goal — for that you reach for LangChain or `aisuite`).
Bedrock and Vertex are supported via the same `AWS_*` /
`GOOGLE_APPLICATION_CREDENTIALS` env vars `claude-code` honors.

## 4. MCP support

**Yes — first-class.** `ClaudeAgentOptions(mcp_servers=...)` takes the
same MCP server config schema as the Node CLI. You can also register
*in-process* tools (decorated Python functions) that show up to Claude
as MCP-style tools without a separate server process — a clean path to
"give my agent these three Python helpers and that one external MCP
server."

## 5. Sub-agent model

**Yes.** The SDK exposes Claude Code's `Task` sub-agent primitive: a
parent session spawns child sessions with their own system prompt, tool
allowlist, and turn budget; results stream back into the parent's
context. This is the same machinery `claude-code` uses for its
`subagent` configuration — now scriptable from Python.

## 6. Telemetry stance

**Off by default; opt-in.** The SDK doesn't ship its own analytics. The
underlying Node CLI it invokes follows `claude-code`'s policy
(opt-out telemetry on the OAuth / Pro path; off on raw API-key path,
which is the default for SDK users). Set
`DISABLE_TELEMETRY=1` for belt-and-braces.

## 7. Prompt-cache strategy

**Yes — automatic.** Sessions emit `cache_control: {"type":
"ephemeral"}` markers on the system prompt and on long tool-result
blocks above a size threshold, identical to the Node CLI's behavior.
You can override the cache strategy per turn via
`ClaudeAgentOptions(extra_args={"cache_system_prompt": False})` if you
want to A/B with vs. without. Cache hit/miss counters surface in the
streaming `usage` events.

## 8. Hot APIs (this is a library — there are no keybinds)

The library is async-first. The two main entry points:

```python
from claude_agent_sdk import query, ClaudeAgentOptions

# One-shot, streaming
async for chunk in query(prompt="refactor utils.py", options=ClaudeAgentOptions(
    cwd="/path/to/repo",
    allowed_tools=["read_file","write_file","run_shell"],
    model="claude-sonnet-4-5",
)):
    print(chunk)
```

```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

# Long-lived session with sub-agents + MCP
async with ClaudeSDKClient(options=ClaudeAgentOptions(
    mcp_servers={"github": {"command": "npx", "args": ["-y", "github-mcp-server"]}},
    subagents=[{"name": "reviewer", "system_prompt": "Strict code reviewer", "allowed_tools": ["read_file"]}],
)) as agent:
    await agent.send("Open issue #42, propose a fix, and have the reviewer subagent critique it.")
    async for evt in agent.events():
        ...  # tool_use, tool_result, text, usage, sub_agent_start, ...
```

| Surface | Purpose |
|---------|---------|
| `query(...)` | One-shot streaming generator |
| `ClaudeSDKClient` | Long-lived session w/ MCP + sub-agents + hooks |
| `@tool` decorator | Register an in-process Python function as an MCP-style tool |
| `ClaudeAgentOptions(allowed_tools=, denied_tools=)` | Tool allowlist / denylist |
| `ClaudeAgentOptions(hooks=)` | Pre/post tool-call interception (audit, mutate, refuse) |
| `ClaudeAgentOptions(permission_mode=...)` | `default` / `acceptEdits` / `bypassPermissions` |

## 9. Killer feature, weakness, when to choose

- **Killer:** **the entire `claude-code` runtime — sub-agents, MCP,
  hooks, sandboxed shell, file-edit-with-diffs — exposed as a Python
  library you can embed in *your* CLI**, not a binary you have to
  shell out to and parse stdout from. If you're building a domain
  agent (a release-bot, a code-review CLI, an issue-triager) and you'd
  otherwise be hand-rolling a tool loop on top of `anthropic.Anthropic`,
  this gets you the production-grade harness in 50 LOC.
- **Weakness:** **Claude-only and Node-on-the-side.** Your greenfield
  agent inherits a hard dependency on `node >= 18` and the
  source-available Node CLI for the sandboxed exec layer — that's a
  meaningful packaging tax for a "Python SDK." Also: the API surface
  is still pre-1.0 and renamed once already (was
  `claude-code-sdk`); breaking changes between 0.1.x minors are
  expected. Pin tightly.
- **Choose it when:** you want to ship a Claude-specific in-terminal
  agent and you specifically want sub-agents + MCP + hook-style
  policy enforcement without writing the loop. Pick raw
  [`anthropic`](https://pypi.org/project/anthropic/) SDK if you only
  need single-call chat completion; pick
  [`openai-agents-python`](../openai-agents-python/) if you want the
  same shape but on OpenAI's models; pick
  [`langgraph`](../langgraph/) if you need multi-provider and an
  explicit graph DSL more than you need Claude-shaped sub-agents.

## Pitfalls observed in real use

1. **Forgot the Node dependency.** `pip install claude-agent-sdk`
   succeeds without `@anthropic-ai/claude-code` being on PATH. The
   first call that needs `run_shell` / `write_file` then fails at
   runtime with a confusing `FileNotFoundError: claude`. Make
   `npm i -g @anthropic-ai/claude-code` part of your install docs.
2. **Permission mode default is `default` — meaning every file-edit /
   shell call prompts.** In a script (no TTY) the prompt hangs.
   For headless use set
   `permission_mode="acceptEdits"` and pair with a strict
   `allowed_tools` allowlist. Never ship `bypassPermissions` to a
   shared CI runner.
3. **Sub-agent token usage is billed to the parent session but
   reported separately** in the streaming `usage` events. If you sum
   only `parent` usage your cost dashboard under-reports by exactly
   the sub-agent share. Aggregate `usage` across `sub_agent_*` events
   too.
4. **MCP server `command:` is invoked from `cwd`, not from the
   user's home.** Misconfigured paths (`./node_modules/.bin/...`) work
   in dev and break in prod where `cwd` is `/`. Use absolute paths or
   `npx -y` invocations.
5. **The 0.1.x → 0.1.x minor bumps have shipped renamed kwargs**
   (`system_prompt` ↔ `system`, `tools` ↔ `allowed_tools`). Pin to an
   exact version (`claude-agent-sdk==0.1.68`) until 1.0 lands; a
   `>=0.1` constraint will break your build under you.
6. **In-process `@tool`-decorated Python functions run in the same
   event loop as the agent.** A blocking `time.sleep(30)` inside one
   freezes the streaming surface. Keep tool functions `async`, or use
   `asyncio.to_thread` for genuinely-blocking work.
7. **`hooks` fire on the parent *and* sub-agents** unless you scope
   them explicitly. A "log every tool call to S3" hook installed at
   the parent will double-log calls that sub-agents also make. Filter
   on `event.session_id` if you want per-agent attribution.
