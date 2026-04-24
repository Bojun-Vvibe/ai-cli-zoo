# oatmeal

> Snapshot date: 2026-04. Upstream: <https://github.com/dustinblackman/oatmeal>
> Latest release: `v0.13.0` (2024-03-16). License file:
> [`LICENSE`](https://github.com/dustinblackman/oatmeal/blob/main/LICENSE).

A Rust-built terminal chat TUI for LLMs whose distinguishing feature is
**bidirectional editor integration**: a code block in the chat can be sent
straight into the buffer of a running Neovim / VS Code session, and the
selection in your editor can be pulled back into the chat as a fenced block —
no manual copy-paste, no clipboard race.

## 1. Install footprint

- Single static Rust binary. Distribution channels are unusually wide for a
  hobby-scale project: Homebrew (`brew install dustinblackman/tap/oatmeal`),
  apt repo, dnf repo, Nix (`nur.repos.dustinblackman.oatmeal`), Arch AUR,
  Alpine `.apk`, Chocolatey, Scoop, Docker image, `cargo install oatmeal`,
  or a release tarball.
- Roughly 8–12 MB on disk depending on platform. No daemon, no Python.
- macOS, Linux (x86_64 + arm64), Windows all first-class.
- Config: `$XDG_CONFIG_HOME/oatmeal/config.toml`. Sessions persist to
  `$XDG_DATA_HOME/oatmeal/sessions/`.

## 2. License

MIT. See [`LICENSE`](https://github.com/dustinblackman/oatmeal/blob/main/LICENSE).

## 3. Models supported

Backends are pluggable. Built-in:

- **OpenAI** (any chat-completions endpoint, including Azure OpenAI and any
  OpenAI-compatible proxy via `openai.url`).
- **Ollama** (any model `ollama list` reports — Llama 3.x, Qwen 2.5-Coder,
  DeepSeek-Coder, etc.).
- **`langchain`** backend that bridges into the Langchain ecosystem for the
  long tail (Anthropic, Bedrock, Vertex, Cohere, etc.).

Backend selection is one CLI flag (`--backend ollama`) or one config line.
There is no LiteLLM gateway; multi-provider use means picking the matching
backend per session, not transparent routing.

## 4. MCP support

**No.** Oatmeal predates the broad MCP rollout and the project has been
quiet since v0.13.0 (2024-03). Tool use happens through the editor
integration channel (push/pull a buffer) rather than through a Model
Context Protocol surface.

## 5. Sub-agent model

None. One conversation, one backend, one model at a time. The "agency"
surface is the **editor pipe**: the model can suggest an edit, you accept
the block, the editor updates — but there is no autonomous file-write
loop and no spawned worker process.

## 6. Telemetry stance

**Off.** No analytics, no phone-home, no opt-in flag. The configured
backend (OpenAI endpoint, local Ollama, etc.) sees prompts; nothing else.

## 7. Prompt-cache strategy

None at the application layer. Oatmeal forwards plain chat-completions
calls to whichever backend you pointed it at, so any caching is whatever
that endpoint provides (Anthropic ephemeral cache via an OpenAI-compatible
proxy, OpenAI's automatic prefix cache, etc.). Sessions are persisted to
disk so you can resume a long chat without resending — but the resend on
resume is a fresh prompt to the model, not a cached continuation.

## 8. Hot keybinds (TUI)

| Key | Action |
|-----|--------|
| `Tab` / `Shift+Tab` | Cycle focus between input box and chat history |
| `Enter` | Send prompt |
| `Ctrl+O` | Open the slash-command picker |
| `Ctrl+C` | Cancel an in-flight stream |
| `Ctrl+R` | Regenerate the last assistant message |
| `Ctrl+B` | Send the current code block to the connected editor |
| `Ctrl+Y` | Copy the current code block to system clipboard |
| `q` (in chat focus) | Quit |

Slash commands include `/append <file>` (read a file into the next prompt),
`/copy`, `/quit`, and `/model <name>` for in-session model switching.

## 9. Killer feature, weakness, when to choose

- **Killer:** the **editor bridge**. With the Neovim plugin running you can
  yank a visual selection straight into the Oatmeal chat as a fenced block,
  and accept the model's reply directly back into the buffer at the cursor
  — chat and editor stay in lockstep without the clipboard middleman that
  breaks indentation and trailing newlines. VS Code integration is the
  same pattern via a companion extension.
- **Weakness:** project has been **inactive since 2024-03** (no release in
  over two years as of this snapshot). No MCP, no agent loop, no
  Anthropic-native backend (you reach Claude via an OpenAI-compatible
  proxy or via `langchain`). Treat as a battle-tested classic, not a
  living roadmap.
- **Choose it when:** you live in Neovim or VS Code, you want a chat TUI
  that talks fluidly to *that* editor without becoming an agent that
  rewrites files behind your back, and you are happy with OpenAI / Ollama
  / Langchain as your model surface. Skip if you need MCP, true sub-agents,
  or active upstream development.
