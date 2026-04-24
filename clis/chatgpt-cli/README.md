# chatgpt-cli

> Snapshot date: 2026-04. Upstream: <https://github.com/kardolus/chatgpt-cli>
> Last verified version: **v1.10.11** (2026-03-22). License file:
> [`LICENSE`](https://github.com/kardolus/chatgpt-cli/blob/main/LICENSE) — MIT.

A single Go binary that started as a thin OpenAI chat wrapper and
has, over five years, grown into a multi-provider LLM workbench:
streaming chat, named "thread" sessions on disk, prompt files,
image / audio I/O, MCP tool calls, and an experimental **agent mode**
(ReAct + Plan/Execute) with budget caps and a workdir sandbox.

It overlaps in surface with [`aichat`](../aichat/) and
[`mods`](../mods/), but the differentiator is the agent mode: this
is one of a small number of CLIs that exposes a **multi-step
ReAct/Plan-Execute loop with explicit cost and policy controls** as a
first-class feature, not a bolt-on.

## 1. Install footprint

- `brew install kardolus/chatgpt-cli/chatgpt-cli` (macOS / Linux).
- Pre-built binaries on the [releases](https://github.com/kardolus/chatgpt-cli/releases)
  page for macOS (Apple Silicon + Intel), Linux (amd64 / arm64 / 386),
  FreeBSD (amd64 / arm64), Windows (amd64).
- Single static Go binary, ~12 MB. Zero runtime deps.
- Config in `~/.chatgpt-cli/config.yaml` (or
  `$CHATGPT_CLI_CONFIG_HOME`). Per-provider API keys via env vars
  (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `PERPLEXITY_API_KEY`, etc.)
  or in the config file.
- Threads (named conversation histories) live as JSON under
  `~/.chatgpt-cli/history/`.

## 2. Repo, version, license

- Repo: <https://github.com/kardolus/chatgpt-cli>
- Version checked: **v1.10.11** (2026-03-22). Project ships
  small releases roughly monthly.
- License: MIT. License file at the repo root:
  [`LICENSE`](https://github.com/kardolus/chatgpt-cli/blob/main/LICENSE).
- Homebrew tap: `kardolus/chatgpt-cli`.

## 3. What it actually does

Five overlapping surfaces from one binary:

- **Query mode**: `chatgpt "your question"` — one-shot input, one
  streamed reply, exits.
- **Interactive mode**: `chatgpt --interactive` — REPL with
  thread-scoped history (`--thread <name>` switches threads;
  histories persist across sessions and survive restarts).
- **Pipe mode**: `cat foo.txt | chatgpt "summarize"` — stdin becomes
  the user message; the standard Unix-pipe shape that
  [`mods`](../mods/) popularised.
- **Prompt files**: `chatgpt --prompt review-go` reads a named prompt
  file from `~/.chatgpt-cli/prompts/` (or the bundled prompt
  catalog) and uses it as the system prompt — a "named persona"
  pattern similar to [`fabric`](../fabric/) but lighter-weight.
- **Agent mode**: `chatgpt --agent "<task>"` runs a ReAct or
  Plan/Execute loop with shell, file, and LLM-reasoning tools,
  bounded by a workdir sandbox and per-task budget (max steps, max
  tokens, max USD).

Agent-mode safety is real, not cosmetic:

- **Workdir safety**: every tool call is confined to a configured
  workdir; attempts to read or write outside it are rejected at the
  tool layer, not just discouraged in the prompt.
- **Budget caps**: `max_steps`, `max_tokens`, and `max_cost_usd` are
  hard limits — when any one trips, the agent stops and reports.
- **Default policy**: by default, destructive shell commands
  (`rm -rf`, `git push -f`, etc.) require explicit allowlisting in
  the policy section of the config.
- **Logs**: every step (thought, action, observation) is journaled to
  a per-task log file under `~/.chatgpt-cli/agent-logs/` for after-
  the-fact audit.

## 4. MCP support

Yes — client. Configure MCP servers (stdio or HTTP/SSE) in the
config under `mcp_servers:`. Tools exposed by those servers become
callable in both interactive mode and agent mode. Headers and bearer
tokens are configurable per server. Session reuse across calls is
supported. The README has worked examples for filesystem, fetch, and
custom HTTP MCP servers.

It does **not** ship an MCP server itself.

## 5. Sub-agent model

Agent mode supports two strategies:

- **ReAct**: the canonical Thought → Action → Observation loop, one
  model call per step.
- **Plan/Execute**: a planner pass produces an N-step plan, then an
  executor pass runs each step (potentially with its own tool calls).
  The two passes can use different models (e.g. plan with
  `gpt-5-thinking`, execute with `gpt-4.1-mini`).

There is no role-based multi-agent orchestra — no "reviewer agent"
that critiques an "implementer agent". Plan/Execute is the closest
the tool comes to sub-agents and it is a two-phase strategy, not a
sibling-agent pool.

## 6. Telemetry stance

Off. No analytics in the binary. The configured provider sees
prompts. The agent-mode log files stay local. Cost is computed
locally from per-provider price tables in the binary, not by phoning
home.

## 7. Token / context strategy

- **Sliding window**: per-thread history auto-trims to fit a
  configurable `context-window` token budget; oldest turns drop
  first, system prompt is preserved.
- **Token counts** are computed locally with per-provider tokenizers
  (OpenAI `tiktoken`, Anthropic, Llama tokenizers).
- **Custom context from any source**: pipe stdin or use
  `--context-file`; the file is prepended as a user message and
  counts against the window.
- Agent mode adds the `max_tokens` budget cap on top of the sliding
  window — the agent stops if cumulative tokens exceed the cap, even
  if the window itself still has room.

## 8. Hot keybinds

In interactive mode: standard readline (`Ctrl-A`, `Ctrl-E`,
`Ctrl-R`, `Ctrl-C` to interrupt streaming, `Ctrl-D` to exit). No
custom shortcuts. Tab completion for thread names, prompt names,
and `--target` config keys is shipped via a separate
`chatgpt --completion bash|zsh|fish` generator.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Agent mode with explicit budget + workdir +
policy + log discipline** in a single static Go binary. Other
catalog entries with agent loops either bury the limits in a YAML
file ([`forge`](../forge/)), tie them to a sandbox container
([`OpenHands`](../openhands/)), or omit them entirely. chatgpt-cli's
agent mode is the easiest one in the catalog to *audit after the
fact* — `cat ~/.chatgpt-cli/agent-logs/<task>.log` and you have the
full ReAct trace, token counts, and final USD spend.

**Weakness.** The name is a footgun: in 2026 a tool called
"chatgpt-cli" is easy to misread as OpenAI-only or a marketing
wrapper, when it is in fact a multi-provider workbench. Discovery
suffers — many catalog entries with smaller scope have more stars.
The agent mode is labelled experimental for good reason: the
Plan/Execute path occasionally produces plans that the executor
cannot follow without re-planning, and the loop does not currently
auto-replan on executor failure (it just stops). The MCP integration
is solid for client use but lacks the depth of dedicated MCP-first
hosts like [`opencode`](../opencode/) or [`goose`](../goose/) (no
discovery UI, no in-session enable/disable).

**When to choose.**
- You want one Go binary that covers chat, pipe-mode, prompt files,
  *and* an agent loop, without juggling three tools.
- You need **per-task budget caps in USD** as a first-class feature
  and you want them enforced, not just printed.
- You want a multi-provider chat tool that is **also** an MCP client,
  without paying the install cost of a full TUI like
  [`opencode`](../opencode/) or [`crush`](../crush/).
- You like named-thread persistence on disk that you can `git`.

**When to skip.**
- You want a TUI, not a line-based REPL → [`crush`](../crush/),
  [`opencode`](../opencode/), [`gemini-cli`](../gemini-cli/).
- Your agent loop needs **role-based sub-agents** (planner +
  implementer + reviewer on different models) → [`forge`](../forge/)
  or [`claude-code`](../claude-code/) sub-agents.
- You want the tool to **edit files in place** as its primary loop
  → [`aider`](../aider/), [`opencode`](../opencode/),
  [`claude-code`](../claude-code/).
- You only need a Unix-pipe LLM filter and nothing else →
  [`mods`](../mods/) is smaller and more focused.

## 10. Compared to neighbors in the catalog

| Tool | Lang | Surface | Agent loop | Per-task budget cap | MCP client |
|------|------|---------|------------|--------------------|------------|
| chatgpt-cli | Go | Query / REPL / pipe / agent | ReAct + Plan/Execute | Yes (steps + tokens + USD) | Yes |
| [aichat](../aichat/) | Rust | REPL / pipe / RAG | No (function-script convention) | No | No |
| [mods](../mods/) | Go | Pipe only | No | No | No |
| [forge](../forge/) | Rust | TUI | Yes (YAML-defined workflows) | Per-step model only | Yes |
| [opencode](../opencode/) | TS | TUI | Yes (Task tool sub-agents) | No (session-level only) | Yes |
| [claude-code](../claude-code/) | TS | TUI | Yes (sub-agents + hooks) | No (Pro/Max plan-level) | Yes |

Decision shortcut:

- "I want a Go binary with chat + an auditable agent loop" →
  `chatgpt-cli`.
- "I want a Rust binary with chat + RAG, no agent" → `aichat`.
- "I want the smallest possible Unix pipe" → `mods`.
- "I want a real TUI" → `crush` / `opencode` / `claude-code`.
