# devon

> Snapshot date: 2026-04. Upstream: <https://github.com/entropy-research/Devon>
> Pinned commit: `8f68f1d7467161562d64138aefee7d26c7e68130` (last push 2025-05).
> Package version: `devon-agent` 0.1.25 on PyPI.
> License: AGPL-3.0 — `LICENSE` at repo root.

An open-source **pair-programming agent** with a Python backend
(`devon_agent`), a terminal UI (`devon-tui`, Ink/React in Node) and an
optional Electron desktop app. It occupies the same niche as `aider`,
`plandex`, and `openhands` — autonomous multi-step file editing in a
local repo — but with two distinguishing choices: a **Plan / Code / Test
loop** the agent explicitly walks, and a TUI that surfaces every shell
command and file edit for per-step approval before execution.

The right pick when you want an `aider`-class agent with a richer
**approval-gated** TUI and you can tolerate AGPL on the agent process.

## 1. Install footprint

- Two installs, one process pair:
  - Backend: `pip install devon-agent` (Python 3.10+) — exposes the
    `devon_agent` HTTP server on a local port.
  - TUI: `npm install -g devon-tui` — Ink/React terminal UI that talks
    to the backend over HTTP.
- Or: `curl -fsSL https://raw.githubusercontent.com/entropy-research/Devon/main/install.sh | bash`
  (one-shot installer that pins both halves).
- Workspace state under `~/.devon/`: session transcripts, plan trees,
  cached repo-map. Per-project config in `.devon.config` at the repo
  root.
- Electron desktop is optional and not required for CLI use.

## 2. License

**AGPL-3.0** for the agent backend (`devon_agent`) and the TUI.
SPDX-detectable at the root `LICENSE`. AGPL applies to the *agent
process*, not to your application code that the agent edits — but if
you fork Devon and run a hosted version, the AGPL network-use clause
kicks in. For internal-only use this is a non-issue.

## 3. Models supported

Backend supports:

- **Anthropic** Claude 3 family (Opus, Sonnet, Haiku) — the default
  recommended model; the planning prompts are tuned for Claude.
- **OpenAI** GPT-4 / GPT-4o / o-series via the OpenAI client.
- **Groq** (`llama-3.x-70b-versatile`) for the fast-iteration loop.
- **Ollama** for fully-local runs (slower; the planner prompts are long
  and 8B-class local models tend to lose the plan).
- **Custom OpenAI-compatible** endpoints (vLLM, llama.cpp server,
  any LiteLLM-fronted backend) by setting `OPENAI_API_BASE`.

Model selection: `devon configure` wizard or `--model` flag on
`devon-tui`.

## 4. MCP support

None as of 0.1.25. Tools are **internal** — file read/write, shell
exec, search, and a planner tool are all hard-coded in the backend.
There is no MCP client to point at external servers and no
`devon --mcp` server to expose Devon's tools. If MCP is mandatory, pick
`goose` or `opencode`.

## 5. Sub-agent model

**Plan-Code-Test loop, single-thread.** The backend orchestrates three
prompts in sequence (plan a step, execute it via tools, run tests /
verify) and updates a shared session state. There is no parallel
sub-agent dispatch — one tool call at a time, surfaced to the TUI for
approval. The "agent" is one model with a tools harness, not a swarm.

## 6. Telemetry stance

**Opt-in.** Backend posts anonymous run metrics (model, run length,
success/failure) to the maintainers' endpoint **only if** you answer
"yes" during `devon configure`. The default in the wizard is "no".
The TUI is fully local; no analytics SDK.

## 7. Prompt-cache strategy

Anthropic prompt caching is set on the Claude backend by default — the
system prompt and the planner's repo-map summary are tagged
`cache_control: ephemeral` so the 5-minute cache TTL covers a typical
multi-step session. For OpenAI / Groq backends there is no equivalent
cache layer; the planner prompt is re-sent in full every step. Local
Ollama runs reuse the KV-cache via `keep_alive`.

The session transcript is the long-term memory, not a cache: it gets
truncated on token-limit overflow with a summarization pass.

## 8. Hot keybinds

`devon-tui` (the Ink terminal UI):

- `Enter` — send the current prompt to the agent.
- `Esc` — interrupt the running step (drops back to the planner).
- `Ctrl-A` — approve the proposed shell command / file edit.
- `Ctrl-D` — deny the proposed action and ask the agent to revise.
- `Ctrl-E` — open the proposed diff in `$EDITOR` for hand-editing
  before the agent applies it.
- `Tab` — switch focus between the chat pane, the diff preview, and
  the file tree.
- `Ctrl-N` / `Ctrl-P` — next / previous step in the current plan.
- `Ctrl-S` — save the current session transcript.
- `Ctrl-Q` — quit (asks for confirmation if the agent is mid-step).

CLI flags on `devon-tui`:

- `devon-tui --model anthropic/claude-3.5-sonnet`
- `devon-tui --port 10000` (override backend port)
- `devon-tui --workspace /path/to/repo`
- `devon-tui --auto` — disable per-step approval; agent runs to completion
  unless interrupted (use only inside a sandbox / VM).

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Approval-gated TUI with editable diffs.** Every
proposed file edit shows up as a unified diff in a side pane; you can
`Ctrl-E` it to edit the diff in your `$EDITOR` before the agent
applies it, or `Ctrl-D` to reject with a free-form reason that goes
back into the planner as a constraint. No other open-source pair
programmer in the catalog combines per-step approval with **mutate the
diff before applying** in this fashion — `aider` lets you reject and
re-prompt, `plandex` lets you preview a branch, but neither hands you
the editor on the unapplied diff.

**Weakness.** **Maintenance is slow.** Last commit on `main` is
2025-05; PyPI version `0.1.25` from the same window. The backend +
TUI two-process split is fragile (port conflicts on `:10000`, version
skew between `pip` and `npm` halves causes silent feature drops).
Newer Claude models work but the planner prompts were tuned for
Claude 3.5 Sonnet and the o-series planning style does not map
cleanly. Test-loop integration assumes a `pytest` / `npm test` -shaped
project; non-standard test runners need manual config.

**When to choose.**

- You want an `aider`-style autonomous file editor with **richer
  per-step UI** and **diff hand-editing** before apply.
- You are comfortable with **AGPL on the agent process** for internal
  use (the AGPL does not infect the code being edited).
- You target **Claude 3.5 Sonnet** as your primary model and want a
  planner already tuned for it.
- You want a **separable backend** (the Python server is reusable from
  custom clients; the official TUI is just one consumer).

**When not to choose.** You need MCP, you need active 2026 release
cadence, you target o-series / GPT-5 as primary, you want a single
binary (the Python+Node split is a real install burden), or you need
parallel sub-agents. Pick `aider` (lighter), `plandex` (long-horizon
plans), `openhands` (sandboxed, multi-agent), or `opencode` (MCP +
plugins) instead.
