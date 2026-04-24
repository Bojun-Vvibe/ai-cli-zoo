# tenere

> Snapshot date: 2026-04. Upstream: <https://github.com/pythops/tenere>
> Latest release: `v0.11.3` (2025-09-01). License file:
> [`LICENSE`](https://github.com/pythops/tenere/blob/master/LICENSE).

A small, sharply-scoped Rust TUI for chatting with LLMs. The pitch is the
opposite of a feature-checklist agent: **vim keybindings, syntax-highlighted
chat bubbles, three backends, and nothing else**. Static binary, no daemon,
no editor bridge, no tool calls — just a fast keyboard-driven chat surface.

## 1. Install footprint

- Single static Rust binary (~5 MB stripped).
- `brew install tenere`, `cargo install tenere`, `nix-env -iA nixpkgs.tenere`,
  or grab a release tarball. Also runs on Android via `nix-on-droid`.
- Linux, macOS, Windows. No runtime deps beyond a system clipboard provider
  (`xsel` / `wl-clipboard` / `pbcopy`) for paste-into-prompt.
- Config: `$XDG_CONFIG_HOME/tenere/config.toml` on Linux,
  `~/Library/Application Support/tenere/config.toml` on macOS. Override with
  `tenere -c <path>`.
- Saved chats are plain `.txt` / `.md` files in `~/.config/tenere/archive/`.
  The last saved chat is auto-loaded on startup unless disabled.

## 2. License

GPL-3.0. See [`LICENSE`](https://github.com/pythops/tenere/blob/master/LICENSE).
Copyleft — incompatible with closed-source bundling.

## 3. Models supported

Three first-party backends, selected via the `llm` config key:

- **`chatgpt`** — OpenAI Chat Completions (defaults to `gpt-4`; any model
  accepted by the configured endpoint works, and `url` can repoint it at any
  OpenAI-compatible proxy).
- **`llamacpp`** — `llama.cpp` server (`llama-server`) over its
  OpenAI-compatible HTTP surface.
- **`ollama`** — local Ollama daemon (any model `ollama list` reports).

There is no Anthropic / Gemini / Bedrock backend, no LiteLLM gateway, and no
plugin system. To reach Claude or Gemini you point the `chatgpt` backend at
a proxy that translates (LiteLLM, OpenRouter, or similar).

## 4. MCP support

**No.** Tenere is intentionally a chat surface, not a tool-using agent;
there is no MCP client, no shell-tool, no file-edit tool. Conversations
stay text-in / text-out.

## 5. Sub-agent model

None. One conversation, one model. Multiple chats can be open as separate
tabs (`Ctrl+t` to create, `Ctrl+w` to close, `Tab` to cycle), but each tab
is its own independent stream — no parent/child agency, no delegation.

## 6. Telemetry stance

**Off.** No analytics, no opt-in flag. The configured endpoint
(OpenAI / `llama.cpp` server / Ollama) sees prompts; nothing else.
Archive files stay on local disk.

## 7. Prompt-cache strategy

None at the application layer. Whatever the configured endpoint provides
(OpenAI's automatic prefix cache, llama.cpp's KV-cache reuse on consecutive
requests, Ollama's `keep_alive` window) is what you get. Tenere itself
ships the full chat history every turn — no client-side trimming, no
summarization pass.

## 8. Hot keybinds (TUI)

Modal, vim-flavoured. `Esc` enters Normal mode; `i` returns to Insert mode
in the prompt box.

| Mode  | Key | Action |
|-------|-----|--------|
| Insert | `Enter` | Send prompt |
| Insert | `Alt+Enter` / `Ctrl+M` | Newline inside the prompt (multi-line input) |
| Normal | `i` / `a` | Insert mode at / after cursor |
| Normal | `dd` | Clear the prompt buffer |
| Normal | `y` | Yank the prompt buffer to clipboard |
| Normal | `p` | Paste clipboard into the prompt |
| Normal | `gg` / `G` | Jump to top / bottom of chat history |
| Normal | `Ctrl+n` | New chat (archives current) |
| Normal | `Ctrl+s` | Save the current chat to a file |
| Normal | `Ctrl+h` | Open chat history picker |
| Normal | `Ctrl+t` | New tab |
| Normal | `Ctrl+w` | Close current tab |
| Normal | `Tab` / `Shift+Tab` | Cycle tabs |
| Any   | `Ctrl+c` | Cancel in-flight stream |
| Normal | `q` | Quit |

## 9. Killer feature, weakness, when to choose

- **Killer:** the **modal vim editing inside the prompt buffer** plus
  multi-tab independent chats inside one process, with chat persistence
  and auto-restore — all behind a single ~5 MB static binary that boots
  instantly. No other TUI in the catalog combines real vim modal editing
  on the prompt with tabs and crash-resume.
- **Weakness:** **no agency, no tools, no MCP, no Anthropic / Gemini
  backend.** Reaching Claude or Gemini means standing up an OpenAI-compatible
  proxy (LiteLLM, OpenRouter). GPL-3.0 license blocks commercial redistribution
  in proprietary products. Backend matrix is narrower than [`elia`](../elia/)
  or [`oatmeal`](../oatmeal/).
- **Choose it when:** you want a Vim-feeling chat REPL with persistent
  sessions and tabs, you live in OpenAI / Ollama / `llama.cpp` already,
  and you explicitly *do not* want the model to touch your filesystem.
  Pick [`elia`](../elia/) instead if you need Anthropic / Gemini / Groq
  natively, [`oterm`](../oterm/) if you want the same vibe but
  Ollama-only with MCP, or [`oatmeal`](../oatmeal/) if you want chat
  glued to a running Neovim / VS Code session.
