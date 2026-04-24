# shell-ai

> Snapshot date: 2026-04. Upstream: <https://github.com/ricklamers/shell-ai>

A Python CLI (`shai`) that turns a plain-English description into
**N candidate shell commands** and lets you arrow-key one. Sits in
the same niche as `shell-gpt`, `shell-genie`, `gorilla-cli`,
`ai-shell`, `tlm`, `cmdh`, `yai` — but its differentiator is a
LangChain backend that switches between OpenAI / Azure OpenAI /
Groq / Ollama / Mistral by changing one env var, plus a built-in
multi-suggestion picker (default 3, configurable via
`SHAI_SUGGESTION_COUNT`).

## 1. Install footprint

- `pip install shell-ai`. Python 3.10+ on Linux, 3.9+ elsewhere.
- Pulls LangChain + the OpenAI / Groq / Mistral / Ollama provider
  packages it needs (~50 MB resolved).
- Config at `~/.config/shell-ai/config.json` (Linux/macOS) or
  `%APPDATA%\shell-ai\config.json` (Windows).
- No daemon, no shell hook. The binary is `shai`.

## 2. Repo + version + license

- Repo: <https://github.com/ricklamers/shell-ai>
- Latest published `setup.py` version: **v0.4.4** (no GitHub
  releases tagged; PyPI is the source of truth).
- License: **MIT** —
  <https://github.com/ricklamers/shell-ai/blob/main/LICENSE>
- Default branch: `main`.

## 3. Models supported

`SHAI_API_PROVIDER` selects one of `openai` (default), `azure`,
`groq`, `ollama`, `mistral`. Anything OpenAI-compatible can be
reached by setting `OPENAI_API_BASE` while keeping
`SHAI_API_PROVIDER=openai` (so vLLM, LM Studio, OpenRouter,
DeepSeek, LiteLLM proxy all work). Per-provider model env vars
(`OPENAI_MODEL`, `OLLAMA_MODEL`, etc.) pick the actual model.

## 4. MCP support

None. The tool is a one-shot translator: prompt in, N candidates
out. There is no agent loop and no tool surface for an MCP server
to plug into.

## 5. Sub-agent model

None. One LLM call returns a JSON list of N candidates, the TUI
renders them with InquirerPy. Selection runs the chosen command
in your shell and writes it to history (unless
`SHAI_SKIP_HISTORY=true`).

## 6. Telemetry stance

Off, no analytics. Egress is exclusively to the configured LLM
provider endpoint. The selected command lands in your shell
history file by default — disable with `SHAI_SKIP_HISTORY=true`.

## 7. Prompt-cache strategy

None. Each `shai run "..."` is an independent LangChain call; no
client-side cache, no provider prefix-cache hinting. The prompts
are short enough that this rarely matters.

## 8. Hot keybinds / commands

- `shai run "<intent>"` — generate suggestions, arrow-key picker,
  Enter to execute (or Esc to cancel).
- `shai run -n 5 "..."` — override `SHAI_SUGGESTION_COUNT`
  inline.
- `SHAI_SKIP_CONFIRM=true shai run "..."` — auto-execute the top
  candidate (DANGER: skips the picker entirely).
- `CTX=true shai run "..."` — context mode: pipes recent shell
  output into the prompt so the model can disambiguate (e.g.
  "rerun that but with `--dry-run`").

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Provider-mix as one env var.** Most peers
in the niche are pinned to one model family or require a config
edit to switch. `SHAI_API_PROVIDER=groq` flips the same binary
from OpenAI to Groq's free tier; `SHAI_API_PROVIDER=ollama` flips
it to a local model. Combined with the default 3-candidate
arrow-key picker, you get the `aicommits -g` UX for shell
commands without the OpenAI lock-in `aicommits` has.

**Weakness.** **LangChain dependency cost.** The install footprint
and import time are heavier than the single-binary peers
(`tlm`, `tgpt`, `clai`). Cold start measurably noticeable
(~600 ms) compared to `tgpt` (~30 ms). Also: no shell-hotkey
integration (`Ctrl-L`-style) like `shell-gpt` ships — you have
to type `shai run` each time.

**When to choose.**

- You want a **multi-candidate picker** (`-g`-style) for shell
  commands and refuse to pin to one provider.
- You already use **Groq** or **Mistral** as your default
  inference and want shell suggestions on the same key.
- You want **context-aware** suggestions (`CTX=true`) — the model
  sees recent shell output, not just the bare intent.

**When not to choose.** Cold-start matters
(→ [`tgpt`](../tgpt/), [`tlm`](../tlm/)). You want a `Ctrl-L`
shell-hotkey (→ [`shell-gpt`](../shell-gpt/)). You want an
`[E]xecute / [R]evise / [C]opy` revise loop instead of a picker
(→ [`ai-shell`](../ai-shell/)). You want prerequisite install
commands separated as first-class output
(→ [`cmdh`](../cmdh/)).

## Comparable tools in this catalog

- [`shell-gpt`](../shell-gpt/) — Python peer with `Ctrl-L`
  shell-integration; single answer, OpenAI-shaped only.
- [`gorilla-cli`](../gorilla-cli/) — also a multi-candidate picker;
  hosted endpoint logs prompts by default.
- [`ai-shell`](../ai-shell/) — TS peer with an `[E]xecute /
  [R]evise / [C]opy` iterative loop instead of a picker.
- [`shell-genie`](../shell-genie/) — minimal Python peer with
  shell-aware prompt templates (bash / zsh / fish / PowerShell).
- [`tlm`](../tlm/) — local-only Go binary, Ollama backend.
- [`cmdh`](../cmdh/) — separates prerequisite install commands
  from the desired command as first-class output.
- [`yai`](../yai/) — Bubble Tea TUI fusing exec + chat in one
  Go binary.
