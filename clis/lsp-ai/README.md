# lsp-ai

> Snapshot date: 2026-04. Upstream: <https://github.com/SilasMarvin/lsp-ai>
> Pinned version: **v0.7.1**. License file: [`LICENSE`](https://github.com/SilasMarvin/lsp-ai/blob/main/LICENSE).

A standalone Language Server Protocol implementation that exposes LLM-powered
features (inline completion, code actions, "explain this", "refactor this")
to **any** LSP-aware editor. The pitch: the editor doesn't need a per-vendor
plugin — it just speaks LSP, and `lsp-ai` is the AI backend.

## 1. Install footprint

- Single Rust binary: `cargo install lsp-ai`, or grab a release binary from
  GitHub.
- ~15 MB static binary; no runtime deps beyond your chosen model backend
  (Ollama / llama.cpp / Anthropic / OpenAI keys).
- Configured per-editor via that editor's normal "register a language
  server" mechanism (Helix `languages.toml`, Neovim `nvim-lspconfig`,
  VS Code `settings.json`, Emacs `eglot`/`lsp-mode`, etc.).

## 2. License

**MIT**. License text at [`LICENSE`](https://github.com/SilasMarvin/lsp-ai/blob/main/LICENSE).
Permissive enough to embed in commercial editors or proprietary internal
tooling without ceremony.

## 3. Models supported

- Anthropic Claude (Sonnet / Haiku / Opus).
- OpenAI GPT-4 / GPT-4o / GPT-4.1.
- Mistral Codestral and the Mistral chat models via the Mistral provider.
- Local: llama.cpp (built in), Ollama, any OpenAI-compatible HTTP endpoint.
- Models are declared as a **named pool** in the config, then individual
  features (`completion`, `chat`, `action`) reference a pool by name — so
  you can route inline completion to a fast local model and "refactor" to
  Claude in the same session.

## 4. MCP support

No. The integration boundary is the **editor**, not MCP — `lsp-ai` is itself
the protocol layer the editor talks to. Tool-call / external-system
integration is delegated to whichever model backend you point it at.

## 5. Sub-agent model

None. `lsp-ai` is a backend, not an agent loop. Every LSP request is one
shot: editor sends "complete at this position with this prefix/suffix",
`lsp-ai` constructs a prompt (FIM template if the model supports it) and
returns one completion.

## 6. Telemetry stance

**Off.** No analytics endpoint. Egress is exactly the LLM provider you
configured; binary listens on stdio (the LSP transport) and never opens an
outbound socket on its own.

## 7. Prompt-cache strategy

For chat / action features it relies on the upstream provider's prompt cache
(Anthropic, OpenAI). For inline completion, the templating layer keeps the
system + repo-context block stable across keystrokes so cache hits remain
high even as the cursor moves.

## 8. Hot keybinds (TUI / REPL)

N/a — there is no UI. The bindings live in your editor: in Helix, the
default `space-a` action menu surfaces `lsp-ai`'s code actions; in Neovim,
`vim.lsp.buf.code_action()` does the same; inline completion shows up
through your editor's normal ghost-text path.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Editor-agnostic AI through the protocol every modern
editor already speaks. Switch from VS Code to Helix to Neovim to Emacs and
your AI completion config moves with you — only the "register this LSP"
glue is editor-specific, the model config / prompt templates / feature
toggles are one shared TOML file.

**Weakness.** No agent loop, no sub-agents, no project-wide planning. It
is strictly a per-buffer assistant. For "edit fifteen files to add this
feature", reach for [`aider`](../aider/), [`opencode`](../opencode/), or
[`continue`](../continue/) instead.

**When to choose.** You live in Helix or Neovim, you want one config file
that gives you AI completion + chat in every editor you use, and you don't
want to be on a vendor's plugin upgrade treadmill. Pair it with
[`aider`](../aider/) for the multi-file work and you cover both ends.
