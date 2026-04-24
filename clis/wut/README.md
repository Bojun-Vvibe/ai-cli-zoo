# wut

> Snapshot date: 2026-04. Upstream: <https://github.com/shobrook/wut>

A one-word terminal assistant. You run a command, it does something
confusing, you type `wut`, and an LLM reads the *actual scrollback of
your current pane* and explains what just happened. Optionally with a
follow-up question (`wut "how do I add this to PATH?"`).

It is the smallest possible "explain my last error" CLI in the
catalog. There is no agent, no plan, no edit loop. The unique
ingredient is that it pulls context from `tmux` or `screen` capture
buffers — so it sees the same bytes you saw, not a re-run.

## 1. Install footprint

- `pipx install wut-cli` (recommended).
- Pure Python, ~5 MB venv plus the OpenAI / Anthropic / Ollama SDK
  you actually use.
- **Hard requirement: must be invoked from inside a `tmux` or
  `screen` session.** Without one of those, `wut` cannot read the
  prior pane output and refuses to run. (The roadmap notes a hope
  to drop this; as of v0.1.0 it is mandatory.)
- API keys are read from environment variables — no config file:
  - `OPENAI_API_KEY` or
  - `ANTHROPIC_API_KEY` or
  - `OLLAMA_MODEL=<served-model-name>` for fully-local operation.
- Optional knobs: `OPENAI_MODEL` (default `gpt-4o`),
  `OPENAI_BASE_URL` (default unset; point at any OpenAI-compatible
  endpoint — LiteLLM, vLLM, OpenRouter, Groq).

## 2. Repo, version, license

- Repo: <https://github.com/shobrook/wut>
- Version checked: **v0.1.0** (released 2024-12-15; still the latest
  tag as of 2026-04).
- License: MIT. License file at the repo root: `LICENSE`.
- PyPI package: `wut-cli`.

## 3. What it actually does

On invocation, `wut` asks `tmux` (or `screen`) for the current pane's
scrollback, slices off the last command and its output, formats that
as the user message of a single chat completion, and streams the
reply back to your terminal. That is the whole loop.

If you pass an argument, it is appended as a follow-up question:

```
$ brew install pip
... [error noise] ...
$ wut "how do I add this to PATH?"
```

There is no second turn. `wut` is one shot. If you want a follow-up,
you run `wut` again — it re-reads the (now larger) scrollback, which
includes its own previous answer, so the next call has implicit
conversation context for free.

## 4. MCP support

None. It does not consume MCP servers; it does not expose one. Out
of scope for the tool.

## 5. Sub-agent model

None. Single LLM call per invocation.

## 6. Telemetry stance

Off (no analytics in the tool itself). The OpenAI / Anthropic /
Ollama endpoint you point it at sees the **prompt**, which by
construction includes whatever was on your terminal — *including any
secrets you may have just printed*. This is the security trade-off
of a scrollback-reading tool. Use a local Ollama model
(`OLLAMA_MODEL=...`) when the scrollback might contain credentials,
production hostnames, or NDA material.

## 7. Token / context strategy

The scrollback slice is bounded by what `tmux capture-pane -p` /
`screen -X hardcopy` returns — by default the visible pane plus a
small history buffer. There is no chunking and no summarization. If
your last command produced 50k lines of stack trace, `wut` will try
to send all of it; an OpenAI 128k context model handles it, a small
local model will refuse or truncate.

Mitigation: increase `tmux`'s `history-limit` if you want more
context, or `clear && <command-you-want-explained>` immediately
before `wut` to bound the window from below.

## 8. Hot keybinds

None — `wut` exits after one reply. The "keybind" is to alias `wut`
to a short shell key chord in your `.zshrc` / `.bashrc` if you want
single-keystroke invocation.

## 9. Killer feature, weakness, when to choose

**Killer feature.** It is the **only entry in the catalog that uses
your terminal scrollback as the prompt**. Every other "explain my
error" tool in the catalog (`shell-gpt`, `tlm`, `gorilla-cli`,
`ai-shell`, `tgpt`) wants you to either retype the command, paste
the error, or pre-declare what you want explained. `wut` reads what
you actually saw. Three keystrokes, zero copy-paste.

**Weakness.** The `tmux` / `screen` requirement is a hard wall — if
you live in a plain `iTerm2` / `Terminal.app` window, you cannot
use it at all without first wrapping your shell in `tmux`. Single
v0.1.0 release with no version bumps in over a year — feature scope
is intentionally tiny but the roadmap items (drop the multiplexer
requirement, `--fix` mode, Homebrew formula) are all unstarted as
of 2026-04. No plugin system, no model routing beyond
OpenAI/Anthropic/Ollama.

**When to choose.**
- You already live in `tmux` or `screen`.
- You want a "what just happened?" explainer that needs **zero
  ceremony**: no copy, no paste, no flag, no mode.
- You want it to work with a **local Ollama model** so secrets in
  scrollback never leave the host (`OLLAMA_MODEL=llama3 wut`).

**When to skip.**
- You do not use a terminal multiplexer and do not want to start.
- You want the suggestion to be a **runnable command** you can
  execute inline — use [`shell-gpt`](../shell-gpt/) (with `Ctrl-L`
  shell integration) or [`ai-shell`](../ai-shell/) (`E/R/C` revise
  loop) instead.
- You want **multi-turn** debugging dialogue — use a real chat TUI
  ([`elia`](../elia/), [`oterm`](../oterm/), [`aichat`](../aichat/))
  and paste the failing block in.
- You want a tool that will *fix* the problem, not just explain it
  — [`aider`](../aider/) or [`claude-code`](../claude-code/).

## 10. Compared to neighbors in the catalog

| Tool | Input source | Output | Multi-turn | Local-only option |
|------|--------------|--------|------------|-------------------|
| wut | tmux / screen scrollback (auto) | Plain prose explanation | No | Ollama |
| [shell-gpt](../shell-gpt/) | Argument or stdin | Shell command (`Ctrl-L` runs it) | Sessions | OpenAI-compatible |
| [tlm](../tlm/) | Argument | Shell command | No | Ollama-only |
| [gorilla-cli](../gorilla-cli/) | Argument | N candidate shell commands | No | OpenAI-compatible |
| [ai-shell](../ai-shell/) | Argument | Shell command + `[E]/[R]/[C]` revise loop | Single revise | OpenAI-compatible |
| [tgpt](../tgpt/) | Argument | Free-form chat | Flat REPL | Optional providers |

Decision shortcut:

- "Explain whatever just happened in my pane, no copy-paste" →
  `wut`.
- "Translate my intent into a runnable shell command" →
  `shell-gpt` / `tlm` / `gorilla-cli` / `ai-shell`.
- "Have a conversation about the error" → `elia` / `oterm` /
  `aichat`.
- "Fix the underlying bug, not just explain it" → `aider` /
  `claude-code`.
