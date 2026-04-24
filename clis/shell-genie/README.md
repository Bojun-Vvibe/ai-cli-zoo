# shell-genie

> Snapshot date: 2026-04. Upstream: <https://github.com/dylanjcastillo/shell_genie>
> Binary name: `shell-genie`

`shell-genie` is the **smallest entry in the catalog whose entire job
is "natural language → one shell command, then ask"**. It is not a
chat REPL, not a code editor, not a packer, not a planner. You type
`shell-genie ask "find all jpegs larger than 5MB modified this week"`
and you get a single proposed command plus a `Do you want to run the
command? [y/n]` prompt. That is the entire surface.

It overlaps in obvious ways with `shell-gpt -s`, `tgpt -s`, and
`gorilla-cli`, but the niche it fills is distinct: **the simplest,
most boring, lowest-feature shell-command suggester in the catalog**.
No Ctrl-L hook, no multi-candidate picker, no provider matrix, no
session memory. One prompt, one command, y/n. It is the tool you
install for a beginner who would be confused by a TUI.

Three things define the surface:

- `shell-genie ask "<prompt>"` — emits one shell command, prompts to
  run it.
- `shell-genie ask --explain "<prompt>"` — same, but the model also
  prints a one-paragraph explanation of what each piece does before
  the y/n prompt.
- `shell-genie feedback` — writes a structured feedback record to
  `~/.config/shell_genie/feedback.json`, which the project uses to
  curate examples for future prompt tuning. Opt-in only; nothing is
  sent over the network.

## 1. Install footprint

- Pure Python, distributed via PyPI: `pipx install shell-genie`.
  `pip install shell-genie` works in a venv, but `pipx` is the
  documented path because the entry is a global CLI, not a library.
- Two runtime dependencies of note: `typer` (CLI framework) and
  `openai` (used as the HTTP client even when targeting non-OpenAI
  endpoints, by setting `OPENAI_API_BASE`). No native deps, no
  Node, no Rust toolchain.
- Config lives at `~/.config/shell_genie/config.json` and is created
  on first run by an interactive `shell-genie configure` flow that
  asks for backend (`openai` / `free-genie`) and your shell (`bash` /
  `zsh` / `fish` / `powershell`). The shell value matters — the
  generated command is tailored to that shell's syntax (e.g. `fish`
  gets `set` instead of `export`).
- Python 3.9+. Tested on macOS, Linux, Windows; Windows uses the
  PowerShell prompt template when configured for it.

## 2. License

MIT.

## 3. Models supported

Two backends ship in-tree:

- `openai` — uses `OPENAI_API_KEY` and defaults to a chat model
  (currently `gpt-4o-mini`-class; the exact default has shifted with
  upstream releases). You can point at any OpenAI-compatible endpoint
  by setting `OPENAI_API_BASE` — Ollama, vLLM, LM Studio, LiteLLM,
  Groq, OpenRouter, etc. — but the config flow does not expose this,
  you set it as an env var.
- `free-genie` — a hosted free endpoint maintained by the upstream
  author. Rate-limited, no key, intended for "try before you wire up
  a real provider" rather than as a permanent backend.

There is no Anthropic, Gemini, Bedrock, or local-Ollama provider as a
first-class option. If you want those, set `OPENAI_API_BASE` to a
proxy that exposes them in OpenAI shape (LiteLLM does this in one
line), or pick a different CLI from the catalog.

## 4. MCP support

None. There is no MCP client, no MCP server, and no plugin slot.
The whole tool is one file's worth of CLI glue around one chat
completion call.

## 5. Sub-agent model

None. Each invocation is exactly one HTTP round-trip. There is no
"the model ran the command, saw it failed, and tried again" loop —
the model never sees the output. The y/n prompt is the only place a
human stays in the loop. If you want output-aware retry, use
`open-interpreter`, `gptme`, or `codex`.

## 6. Telemetry stance

Off. `shell-genie` itself does not phone home. The optional
`shell-genie feedback` command writes a record locally to
`~/.config/shell_genie/feedback.json` and stops there; uploading it
upstream is a manual, deliberate act (the user copies the file to a
GitHub issue or PR if they want to contribute examples).

The provider you chose obviously sees prompts. With `free-genie`,
treat prompts as you would any anonymous web search; with `openai`
or any other paid endpoint, the provider's policy applies.

## 7. Prompt-cache strategy

None on the client side. The system prompt is short (a few hundred
tokens describing "you generate one shell command for shell X, no
prose, no fences") and is re-sent every call. Provider-side prefix
caching may apply if your endpoint supports it (e.g. Anthropic via
LiteLLM, OpenAI on long static system prompts), but `shell-genie`
does not annotate or version the prompt to optimize for it.

## 8. Hot keybinds

There is no TUI and no REPL, so there are no keybinds in the usual
sense. The post-generation prompt accepts:

- `y` / `Y` / `Enter` — execute the command in your configured shell
  via `subprocess.run` with `shell=True`.
- `n` / `N` — discard and exit.

There is no "edit before run" mode (unlike `tgpt -s`'s `2) Modify`
or `shell-gpt`'s `[D]escribe`). If the suggestion is wrong, you
re-run `shell-genie ask` with a more specific prompt. This is the
most conservative surface in the suggester family: you either run
exactly what was generated, or you do not run it at all.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Aggressive simplicity. There is one command
(`ask`), one config file, one decision (y/n), and one mental model.
For users who will never type `--provider`, `--model`, `-g 3`, or
`Ctrl-L`, `shell-genie` is the lowest-friction way to put an LLM
behind "remember the `find` flags for me". The shell-aware system
prompt (separate templates for bash / zsh / fish / PowerShell) means
generated commands actually parse in the shell the user is in,
which `mods`-style raw-pipe tools do not guarantee.

**Weakness.** Four real ones:

1. **No provider matrix.** First-class support is OpenAI + the free
   endpoint. Anything else requires `OPENAI_API_BASE` and a proxy.
   For a 2026 catalog, that is a narrow set; `tgpt`, `shell-gpt`,
   and `mods` all expose provider flags directly.
2. **No multi-candidate UX.** You get one suggestion. If it is
   wrong, you re-prompt. `gorilla-cli` shows N candidates with arrow
   keys; `aicommits -g 3` does the same for commit messages. For
   shell commands, the multi-candidate UX often beats single-shot.
3. **No edit-before-run.** Unlike `tgpt -s` and `shell-gpt -s`,
   there is no escape hatch to tweak the generated command before
   execution. y or n.
4. **No output feedback loop.** The model does not see whether the
   command worked. If you want "the model retries until the test
   passes", this is the wrong category of tool.

**When to choose.**

- **You are setting this up for a less-technical teammate** and want
  a tool whose entire help text fits on one screen.
- **You already use OpenAI** and do not need a provider abstraction.
- **You want a shell-aware prompt template** without writing one
  yourself — `shell-genie` ships separate templates for bash / zsh /
  fish / PowerShell out of the box.
- **You want a deliberately small dependency** in a personal dotfiles
  repo. One PyPI install, one config file, MIT license.

**When not to choose.**

- You want **multi-candidate selection** → [`gorilla-cli`](../gorilla-cli/).
- You want **`Ctrl-L` shell-buffer integration** → [`shell-gpt`](../shell-gpt/).
- You want **no API key at all** → [`tgpt`](../tgpt/).
- You want **broad provider support out of the box** → [`mods`](../mods/),
  [`llm`](../llm/), [`aichat`](../aichat/).
- You want the model to **see command output and retry** →
  [`open-interpreter`](../open-interpreter/), [`gptme`](../gptme/),
  [`codex`](../codex/).
- You want the model to **edit project files**, not just run shell
  commands → any of the agentic CLIs in the catalog.

## Links

- Upstream: <https://github.com/dylanjcastillo/shell_genie>
- PyPI: <https://pypi.org/project/shell-genie/>
- Author writeup: <https://dylancastillo.co/> (search "shell genie")
