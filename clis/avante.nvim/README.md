# avante.nvim

> Snapshot date: 2026-04. Upstream: <https://github.com/yetone/avante.nvim>

A Cursor-style AI coding experience grafted onto Neovim — diff-based edits,
sidebar chat, and an agent loop with tool use, all driven from inside the
editor you already live in.

## 1. Install footprint

- Neovim 0.10.1+. Install via `lazy.nvim` / `packer` /
  `vim.pack.add({"https://github.com/yetone/avante.nvim"})`.
- Pulls a precompiled Rust shared library (`avante-libs`) on first start —
  templating, tokenizer, and HTTP client live in Rust for speed.
- Latest release: **`v0.0.27`** (the `avante-libs` tag; the Lua plugin
  itself tracks `main` and pins the lib version internally).
- Optional: `ripgrep` for repo grep tool, a Markdown renderer plugin
  (`render-markdown.nvim` or `markview.nvim`) for prettier responses,
  `img-clip.nvim` for paste-image-into-prompt support.

## 2. License

Apache-2.0. See [`LICENSE`](https://github.com/yetone/avante.nvim/blob/6c00a52cb487bb18ea2413dc95a835c7c91db912/LICENSE).

## 3. Models supported

Provider abstraction is first-class:

- Anthropic (Claude Sonnet / Opus / Haiku) — the default.
- OpenAI (GPT-4o / o-series, plus any OpenAI-compatible endpoint).
- Gemini (1.5 / 2.x via Google AI Studio or Vertex).
- AWS Bedrock, Azure OpenAI, Cohere, DeepSeek, Groq, Mistral.
- Local: Ollama, vLLM, LM Studio, llama.cpp server — anything that speaks
  the OpenAI chat-completions schema.
- Per-buffer model override via `:AvanteSwitchProvider`.

## 4. MCP support

**Yes (client).** `mcphub.nvim` integration lets avante consume MCP
servers; tools surface in the agent loop alongside the built-in `view`,
`replace_in_file`, `bash`, and `web_search` tools.

## 5. Sub-agent model

Two modes:

- **`legacy` planning mode** — single agent, plan-then-edit.
- **`agentic` mode** (default in recent versions) — ReAct-style loop with
  parallel tool calls; no true sub-agents but supports an explicit
  *planner* model distinct from the *editor* model (set
  `planner_provider` separately).

## 6. Telemetry stance

**Off.** No analytics wired in. All traffic is direct buffer ↔ LLM
provider; the Rust shared library is the only third-party network user
and only for model API calls.

## 7. Prompt-cache strategy

Anthropic ephemeral cache on the system prompt + project-config block +
the file context section. Long sessions on the same buffer hit the cache
hard. OpenAI uses automatic prefix cache; nothing avante-specific.

## 8. Hot keybinds (default leader is `<leader>a`)

| Binding | Action |
|---------|--------|
| `<leader>aa` | Toggle the avante sidebar (chat) |
| `<leader>ae` | Edit the selected visual region with a prompt |
| `<leader>ar` | Refresh the sidebar from current buffer |
| `<leader>af` | Focus / un-focus the sidebar |
| `<leader>at` | Toggle agent (`agentic`) mode |
| `<leader>a?` | Open the model / provider picker |
| `co` / `ct` / `ca` | (in diff view) choose ours / theirs / accept all |
| `[x` / `]x` | Jump prev / next conflict block in the proposed diff |

## 9. Killer feature, weakness, when to choose

- **Killer:** the **diff UX inside Neovim**. avante doesn't just show you
  a unified diff — it stages the model's proposed edits as a real
  three-way conflict in the buffer, so you use the same `co` / `ct` / `[x`
  / `]x` muscle memory you already use for git merges. Accept-by-hunk is
  trivial. For Neovim users who refuse to leave the editor, this is the
  most native-feeling AI coding loop available.
- **Weakness:** Lua + a precompiled Rust lib means platform support is
  whatever the maintainers built wheels for (mostly Linux x86_64,
  macOS arm64, Windows x86_64); other targets need a local Rust build.
  Sidebar performance can stutter on very long responses on slower
  Neovim builds.
- **Choose it when:** Neovim is your editor, you want a Cursor-grade AI
  loop without leaving it, and you specifically want diff review to feel
  like git-merge resolution rather than a popup approval dialog.
