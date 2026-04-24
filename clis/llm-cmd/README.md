# llm-cmd

> Snapshot date: 2026-04. Upstream: <https://github.com/simonw/llm-cmd>
> Last verified version: **0.2a0** (latest tag). License file:
> [`LICENSE`](https://github.com/simonw/llm-cmd/blob/main/LICENSE) —
> Apache-2.0.

A 100-line plugin for [`llm`](../llm/) that adds a single
subcommand: `llm cmd "..."`. The model proposes a single shell
command for your prompt, prefills it into your readline buffer, and
**lets you edit before pressing Enter**. Then `subprocess` runs it.

Where [`shell-gpt`](../shell-gpt/) shows you `[E]xecute / [D]escribe
/ [A]bort`, where [`gorilla-cli`](../gorilla-cli/) gives you N
candidates with arrow keys, and where [`ai-shell`](../ai-shell/)
gives you `[E]xecute / [R]evise / [C]opy`, `llm-cmd` does the most
unix-native thing: it puts the command on your prompt line as if you
had typed it yourself, and the *normal* line-editor (readline,
zle, fish editor) takes over. You backspace, retype, Ctrl-W,
home/end — exactly the keys you already have muscle memory for.

## 1. Install footprint

- Requires `llm` already installed and configured. Install with
  `llm install llm-cmd`.
- Pure Python, single file (`llm_cmd.py`), depends on `click` and
  `prompt_toolkit` (both pulled in via `llm`'s deps).
- Configuration is whatever `llm` is configured for: model defaults
  via `llm models default <model>`, keys via `llm keys set <provider>`.
  No `llm-cmd`-specific config files.
- macOS, Linux, Windows (WSL or PowerShell) — same surface as `llm`.

## 2. License

Apache-2.0. License file at the repo root: `LICENSE`. Same
license as the host `llm` framework, so the combined install is
unambiguously Apache-2.0.

## 3. Models supported

**Inherits from `llm`.** Whatever `llm` plugins you have installed:
OpenAI built-in, plus Anthropic / Gemini / Mistral / Cohere /
Ollama / OpenRouter / Bedrock / many more via plugins (`llm-claude-3`,
`llm-gemini`, `llm-ollama`, etc.). `llm cmd -m claude-sonnet "..."`
picks the model per call; otherwise `llm models default` applies.

This is the same provider matrix every other `llm` plugin sees —
the substrate is the substrate.

## 4. MCP support

**No.** `llm-cmd` is a single LLM call, no tool loop, no MCP
client. (The host `llm` has experimental `llm-tools-mcp` for tool
use in a different command path, but `llm cmd` does not invoke it.)

## 5. Sub-agent model

**None.** One prompt → one model call → one suggested command.
There is no retry-on-failure, no plan, no follow-up. If the command
fails when you run it, you re-invoke `llm cmd` with a refined
description.

## 6. Telemetry stance

**Off.** Inherits `llm`'s "no analytics, log everything to local
SQLite" posture. Every `llm cmd` invocation is also written to
`~/Library/Application Support/io.datasette.llm/logs.db` (or the
XDG-equivalent path) — searchable later with `llm logs`. The only
network egress is the configured provider.

## 7. Prompt-cache strategy

**None.** Each `llm cmd` invocation is a fresh call with the same
short system prompt — no provider-side `cache_control` markers, no
local prefix cache, no conversation continuation. The system prompt
is small enough that prefix caching wouldn't matter much anyway.

## 8. Hot keybinds (REPL + subcommands)

There is no REPL. The whole UX is one subcommand:

| Invocation | Action |
|------------|--------|
| `llm cmd "<intent>"` | Generate a command, prefill into readline, edit & Enter to execute |
| `llm cmd -m <model> "<intent>"` | Override the default model for this call |
| `llm cmd -s "<system>" "<intent>"` | Override the system prompt (e.g., to bias toward `fish` syntax) |

Once the command is on your prompt line, the keybinds are
**whatever your shell's line editor uses** — readline / zle / fish
editor. That is the entire point of the design: there is nothing
new to learn.

If you abort with Ctrl-C before pressing Enter, no command runs.
If you press Enter, the command goes through `subprocess.run` with
your current shell — there is no second confirmation, no
sandboxing, no `[A]bort` step.

## 9. Killer feature, weakness, when to choose

- **Killer:** **the suggestion lands in your readline buffer, not in
  a TUI dialog.** Every other shell-helper here interrupts your
  flow with a prompt or a menu. `llm-cmd` hands the line to your
  normal editor as if you'd typed it. You skim it, edit the path,
  swap a flag, hit Enter. It's the closest thing to "tab-completion
  for the whole command" rather than "AI assistant overlay."
- **Weakness:** **no safety net at all.** No `[D]escribe`, no
  multi-candidate picker, no sandboxing, no allowlist. If the model
  hallucinates `rm -rf ~` and you press Enter without reading,
  that's a real `rm -rf ~`. The mitigation is "you're supposed to
  read the line before pressing Enter" — which is correct for
  expert users and dangerous for beginners. Also: 0.2a0 means
  "alpha", and the plugin has been at this version since shortly
  after `llm`'s `register_commands` hook stabilized — quiet and
  stable, but explicitly pre-1.0.
- **Choose it when:** you already use [`llm`](../llm/) for chat /
  scripting / SQLite-logged sessions and want one more subcommand
  in the same surface for shell-command suggestions, with editing
  via the line editor you already know. Pick
  [`shell-gpt`](../shell-gpt/) instead if you want
  `[E]xecute/[D]escribe/[A]bort` as a hard step before any command
  runs. Pick [`gorilla-cli`](../gorilla-cli/) if you want N
  candidates with arrow-key selection. Pick
  [`ai-shell`](../ai-shell/) if you want an iterative `[R]evise`
  loop where you nudge the suggestion in natural language without
  retyping intent. Pick [`shell-genie`](../shell-genie/) if you
  want shell-aware (bash/zsh/fish/PowerShell) prompt templates with
  zero flags.

## Pitfalls observed in real use

1. **There is no confirmation step.** Enter executes. This bites
   users coming from `shell-gpt` where the default `[E]xecute`
   prompt acts as a brake. Train yourself to read the line before
   pressing Enter — or don't use `llm-cmd` for destructive
   operations and stick to `shell-gpt` / `gorilla-cli` for those.
2. **The default model matters more than usual.** A weak default
   (e.g., a 3B local model) will hallucinate flags that don't exist.
   Either pin a strong model with `llm models default
   claude-sonnet` (or similar) or pass `-m` per call. The 100-line
   implementation cannot recover from a bad suggestion — it's
   purely a thin wrapper.
3. **Multi-line commands prefill awkwardly.** If the model emits a
   command with embedded newlines (heredocs, multi-statement
   pipelines), prompt_toolkit puts them all on one logical line and
   your shell may not interpret them as you expect. For multi-line
   work, generate to a script with `llm "..."` instead and run the
   script.
4. **Doesn't know your shell.** The system prompt is generic ("you
   are a Linux shell expert" / similar) — it does not detect that
   you're in `fish` or `nu` or PowerShell. If you live in fish,
   pass `-s "Reply with a single fish-shell command, no prose"` or
   wrap in a shell function that prefixes the system prompt. (For
   shell detection out of the box, see
   [`shell-genie`](../shell-genie/).)
5. **The logged prompt in `llm logs` includes the intent string but
   not the executed result.** `llm cmd` doesn't capture stdout /
   stderr / exit code of the command it ran. If you need a
   reproducible "what I asked, what was suggested, what happened",
   wrap in a small shell function that tees output to a log file —
   the plugin doesn't do that for you.
6. **`llm install llm-cmd` requires that `llm` is the same Python
   environment the user actually invokes.** If `llm` is installed
   via `pipx` and you also have a system `llm`, `llm install` will
   land the plugin in the pipx venv only. `which llm` first, then
   `pipx inject llm llm-cmd` (not `llm install`) if you're on
   pipx — easy to get wrong on a fresh Mac.
