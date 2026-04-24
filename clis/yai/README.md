# yai

> Snapshot date: 2026-04. Upstream: <https://github.com/ekkinox/yai>

A Go-binary terminal assistant with **two distinct modes** baked into
a single REPL: ⚡ *exec* mode (natural-language → shell command, with
`[E]xecute / [A]nswer / [C]opy / [X]cancel` confirm prompt) and 💬
*chat* mode (free-form conversation). You toggle between them with
the Tab key inside one running session, so a single `yai` process can
both answer "what's the syntax for `find -mtime`" and offer to run
the resulting command.

It earns a catalog slot for being the **shell-command tool with a
real interactive TUI** (Bubble Tea) rather than the more common
"prompt → suggestion → exit" one-shot shape used by
[`shell-gpt`](../shell-gpt/), [`tlm`](../tlm/), and
[`gorilla-cli`](../gorilla-cli/).

## 1. Install footprint

- One-line installer: `curl -sS https://raw.githubusercontent.com/ekkinox/yai/main/install.sh | bash`,
  or grab the prebuilt binary from the releases page. Single static
  Go binary, ~15 MB. Linux + macOS supported (Windows untested).
- Also installable from source: `go install github.com/ekkinox/yai@latest`.
- First-run wizard writes `~/.config/yai.json` with: OpenAI API key,
  model (default `gpt-3.5-turbo`), preferred response language
  ("english"/"french"/…), preferred shell, OS preset.
- Honors `OPENAI_API_KEY` if pre-set in the environment, otherwise
  prompts during the wizard.

## 2. Repo, version, license

- Repo: <https://github.com/ekkinox/yai>
- Latest release checked: **v0.6.0** (tag `0.6.0`, published
  2023-04-25; project is in maintenance mode with intermittent
  `main`-branch commits).
- License: MIT. License file at the repo root:
  [`LICENSE`](https://github.com/ekkinox/yai/blob/main/LICENSE).
- Go module path: `github.com/ekkinox/yai`.

## 3. What it actually does

Launching `yai` drops you into a Bubble Tea TUI. The header shows
your detected OS, shell, and current mode (⚡ or 💬). Type a request,
hit Enter, and:

- **Exec mode (default)**: yai produces a shell command (one line,
  no markdown fences) plus a short explanation, then shows a confirm
  bar:
  - `[E]xecute` — runs the command in your current shell, captures
    output back into the TUI.
  - `[A]nswer` — flips this single turn into chat mode and explains
    further.
  - `[C]opy` — copies the command to the system clipboard.
  - `[X]cancel` — discards the suggestion.
- **Chat mode**: free-form Q&A, multi-turn, no command extraction.
  Useful for "what does this error mean" follow-ups before going
  back to exec mode for the fix.

Tab toggles modes. `Ctrl-H` opens an in-TUI help panel. `Ctrl-S`
opens a settings panel where you can change the model on the fly.
`Ctrl-L` clears the screen but keeps the session.

One-shot mode is also supported: `yai "list pods sorted by restart
count"` skips the TUI, prints the suggested command + the same
`[E]/[A]/[C]/[X]` line, and exits.

## 4. MCP support

None. `yai` was written before MCP existed and the project has not
adopted the protocol. There are no tool calls beyond "execute the
suggested shell command".

## 5. Sub-agent model

None. Single-LLM, single-thread. Each turn is one chat completion;
chat mode keeps history in memory for the session lifetime; exec
mode is effectively stateless.

## 6. Telemetry stance

**Off — no telemetry in the binary.** Every prompt goes to OpenAI
(or whatever endpoint you configured), which sees the natural-
language request *plus*, in chat-mode follow-ups, captured shell
output you might `[A]nswer` about. No analytics, no update pings,
no remote logging.

## 7. Token / context strategy

- Exec mode is single-turn: each request is a fresh completion seeded
  by a system prompt that bundles your detected OS / shell / preferred
  language and a few-shot template.
- Chat mode keeps the running message list for the session — no
  summarization, no compaction. Long sessions accumulate cost
  linearly; exit and re-enter to reset.
- No cache-control headers, no embedding store, no retrieval. The
  only "memory" between sessions is the `~/.config/yai.json`
  preferences file.

## 8. Hot keybinds

| Key | Action |
|-----|--------|
| `Tab` | Toggle exec ⚡ ↔ chat 💬 |
| `Ctrl-H` | Help panel |
| `Ctrl-S` | Settings panel (change model, language, etc.) |
| `Ctrl-L` | Clear screen, keep session |
| `Ctrl-R` | Reset chat history |
| `Ctrl-C` / `Ctrl-D` | Exit |
| `E` / `A` / `C` / `X` | Confirm-bar actions on a suggested command |

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Exec ⇄ chat in one process.** Most catalog
entries make you choose: command-only translators
([`tlm`](../tlm/), [`shell-gpt`](../shell-gpt/) without `--chat`,
[`gorilla-cli`](../gorilla-cli/)) or chat-only TUIs
([`elia`](../elia/), [`aichat`](../aichat/), [`tgpt`](../tgpt/)). yai
fuses the two with a single Tab key — when the suggested command
doesn't make sense, you `[A]nswer` to flip into a real conversation
without losing the original turn.

**Weakness.** **Project is in maintenance mode.** v0.6.0 (April
2023) is still the latest tagged release; commits land but slowly,
and there's no plugin system, no provider routing beyond OpenAI's
official API, no custom-endpoint support in the config wizard
(though `OPENAI_API_BASE` is honored as an env override).
Single-shell context (auto-detected at launch) means you can't
target a different shell mid-session. No MCP, no tools, no agent
loop.

**Choose it when.**

- You want a **terminal-native chat surface** that *also* knows it
  lives in a shell and can offer to run things — without Python
  install or VS Code dependency.
- You want **single static Go binary** delivery (no venv, no
  Node), `curl | bash` to a working assistant in 30 seconds.
- The Tab-to-toggle exec/chat workflow matches how you actually
  debug: "give me a command" → "wait, why did that fail?" → "OK,
  give me the fixed command" — all in the same TUI.

**Skip it when.**

- You need providers other than OpenAI as first-class — use
  [`mods`](../mods/) (10+ providers, Unix-pipe shape) or
  [`aichat`](../aichat/) (20+ providers + RAG, Rust binary).
- You want a tool that **edits files**, not just runs commands —
  use [`aider`](../aider/), [`opencode`](../opencode/), or
  [`crush`](../crush/).
- You need **persistent searchable history** across sessions — use
  [`elia`](../elia/) (Textual TUI, SQLite history, multi-provider).
- The maintenance-mode pace is a red flag for production reliance.

## 10. Compared to neighbors in the catalog

| Tool | Mode shape | Provider scope | History | Binary delivery |
|------|------------|----------------|---------|-----------------|
| yai | Exec ⇄ chat in one TUI | OpenAI (env-override for compat endpoints) | In-session only | Single Go binary |
| [shell-gpt](../shell-gpt/) | One-shot translator + opt-in `--chat` sessions | OpenAI-compatible | Named on-disk sessions | `pipx` (Python) |
| [tlm](../tlm/) | One-shot suggest/explain/chat | Ollama-only | None | Single Go binary |
| [gorilla-cli](../gorilla-cli/) | One-shot, N candidates | Hosted Gorilla / OpenAI-compatible | None | `pipx` (Python) |
| [ai-shell](../ai-shell/) | One-shot translator with `[E]/[R]/[C]` revise loop | OpenAI-compatible | Optional `ai chat` REPL | `npm -g` (Node) |
| [aichat](../aichat/) | Chat REPL + one-shot, no exec confirm | 20+ providers built-in | Local SQLite | Single Rust binary |

Decision shortcut:

- "I want one TUI for both **'give me the command'** and **'explain
  this error'**, without leaving the keyboard" → `yai`.
- "I want providers other than OpenAI as a first-class concern" →
  `mods` / `aichat`.
- "I want a `[E]xecute / [R]evise / [C]opy` revise loop" →
  `ai-shell`.
- "I want it 100% offline against Ollama" → `tlm`.
