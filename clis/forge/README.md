# forge

> Snapshot date: 2026-04. Upstream: <https://github.com/antinomyhq/forge>

Antinomy's "AI pair programmer for the terminal" — a Rust TUI agent that
leans hard on **multi-provider routing + structured workflows** rather
than locking you into one model. Two things make `forge` distinct in the
zoo: (a) it ships first-class _agent definitions_ as YAML you can fork,
and (b) it treats the model as a hot-swappable backend even mid-session.

This entry exists because most agents in this catalog couple their loop
to one model family's strengths (Claude tools, GPT JSON mode, Gemini
caching). `forge` argues the opposite: keep the loop generic, push the
provider-specific quirks into adapters, let the user pick per task.

## 1. Install footprint

- `npm i -g @antinomyhq/forge` (Node 20+ used as a launcher around the
  Rust binary), or `cargo install forgecode`, or grab a release binary
  from GitHub. macOS / Linux / Windows.
- The Rust binary is ~30 MB; the npm wrapper adds a thin launcher on
  top so `npx forge` works without a Cargo toolchain.
- Per-user state at `~/.forge/` (config, sessions, cached tool schemas).
- Per-project config at `forge.yaml` in the repo root — checked into
  git is the expected workflow, since agent definitions and provider
  routing live there.
- No daemon. Each run is a foreground TUI session or a one-shot
  `forge -p "..."` call.

## 2. License

Apache-2.0.

## 3. Models supported

Provider-agnostic by design. As of v0.x the adapters cover:

- Anthropic (Claude Opus / Sonnet / Haiku, via API key)
- OpenAI (GPT-4.x, o-series, via API key)
- Gemini (1.5 / 2.x, via Google AI Studio)
- OpenRouter (one key, hundreds of models)
- Groq, Together, Fireworks (OpenAI-compatible)
- Local: Ollama, vLLM, llama.cpp server (any OpenAI-compatible endpoint)
- Bedrock and Vertex via the OpenAI-compat adapters with custom base URLs

The killer move: a single session can route different _agent steps_ to
different providers. The "planner" agent runs on Claude Opus, the
"editor" agent runs on a cheap local Qwen model, the "summarizer" runs
on Gemini Flash. You wire this in `forge.yaml`, not in code.

## 4. MCP support

**Yes — client only.** Configure under `mcp:` in `forge.yaml`:

```yaml
mcp:
  servers:
    github:
      command: npx
      args: ["-y", "@modelcontextprotocol/server-github"]
      env:
        GITHUB_PERSONAL_ACCESS_TOKEN: "${GITHUB_TOKEN}"
    filesystem:
      command: npx
      args: ["-y", "@modelcontextprotocol/server-filesystem", "."]
```

Tool schemas from MCP servers are merged into the agent's tool surface
at session start. There is no MCP _server_ mode — `forge` does not
expose its own internals over MCP.

## 5. Sub-agent model

`forge` has the most explicit sub-agent system in this catalog. Agents
are first-class YAML definitions:

```yaml
agents:
  planner:
    model: anthropic/claude-opus-4
    system_prompt_file: ./prompts/planner.md
    tools: [read_file, list_files, grep]
  editor:
    model: openrouter/qwen/qwen-2.5-coder-32b
    system_prompt_file: ./prompts/editor.md
    tools: [read_file, write_file, apply_patch, run_shell]
  reviewer:
    model: anthropic/claude-sonnet-4
    system_prompt_file: ./prompts/reviewer.md
    tools: [read_file, grep]
```

A workflow then chains them:

```yaml
workflows:
  default:
    - agent: planner
    - agent: editor
    - agent: reviewer
      on_reject: retry editor
```

This is closer to a state machine than to a free-form ReAct loop. It
trades flexibility for predictability — useful when you want the agent
to behave the same way across runs and across providers.

## 6. Telemetry stance

**Off by default.** No anonymous usage events ship without explicit
opt-in via `forge.yaml`'s `telemetry: enabled: true`. Crash reports
are also opt-in. The provider adapters obviously talk to whichever
provider you configured — that traffic is on you to audit.

## 7. Prompt-cache strategy

Adapter-dependent:

- Anthropic adapter sets `cache_control: ephemeral` on the system prompt
  and on tool schemas by default.
- OpenAI adapter relies on the provider's automatic prefix caching (no
  explicit hints).
- Gemini adapter uses implicit caching for context > 32k tokens.
- Local providers: no caching, just stateless completions.

There is no cross-provider cache abstraction. Each adapter does what
its backend supports, and `forge` reports cache hit rate in the TUI
status line.

## 8. Hot keybinds (TUI)

- `Ctrl-C` — interrupt the current agent step (does not kill the
  session).
- `Ctrl-D` — exit session, asks to save transcript.
- `Esc` — drop back to the prompt without sending.
- `/agents` — list defined agents and switch.
- `/model <provider/model>` — hot-swap the active agent's model.
- `/tools` — show tool surface for the current agent.
- `/cost` — show running token + dollar cost split by provider.
- `Tab` — cycle through suggested completions for slash commands.

## 9. Killer feature, weakness, when to choose

**Killer feature.** YAML-defined multi-agent workflows where each agent
can target a different provider. You can write a "planner on a frontier
model, editor on a cheap local model" pipeline in 30 lines and have it
behave deterministically across machines. No other entry in the zoo
makes provider-mixing this declarative.

**Weakness.** The YAML surface is large and under-documented in v0.x.
Several config keys exist in the source but not in the README, so
reading the Rust adapter code is occasionally required. The TUI is
also less polished than `crush` or `opencode` — fewer keybinds,
no inline diff preview, no live tool-call animation. Sub-agent
isolation is logical (separate prompts and tool sets) not physical
(no separate process or sandbox), so a misbehaving editor agent can
still trash your working tree.

**When to choose `forge`.**

- You want **provider-mixing as a first-class concept**, not an
  afterthought. Cheap edits, expensive plans, separate keys.
- You want agent behaviour **versioned in git** alongside the code
  it edits. `forge.yaml` + `prompts/*.md` go in the repo, so every
  collaborator runs the same agent.
- You like the **state-machine view** of agent loops more than the
  free-form ReAct view. Workflows are explicit; transitions are
  explicit; failure modes are explicit.

**When not to.** If you want a single-binary one-shot CLI that "just
works" with one model, pick `crush`, `codex`, or `mods` instead. If
you want a polished TUI with skills and hooks, pick `opencode` or
`claude-code`. `forge`'s value is the workflow declarativeness, and
that is overhead you don't need for a five-minute task.
