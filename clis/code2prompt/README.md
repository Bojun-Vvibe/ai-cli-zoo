# code2prompt

> Snapshot date: 2026-04. Upstream: <https://github.com/mufeedvh/code2prompt>

A Rust CLI that walks a directory tree and emits a single prompt-shaped
text blob — source tree header, per-file fenced code, optional Git
metadata, optional Handlebars template wrap, optional clipboard copy,
optional token count. It is the **Rust cousin of `files-to-prompt` /
`repomix`** in the catalog: same job (pack a codebase for an LLM), but
distributed as a single static binary, with first-class **Handlebars
templating** and an interactive **TUI** for picking files.

It is not a model client. It does not call OpenAI, it does not hold a
session, it does not edit files. It emits text. Pipe it into any
catalog entry that takes stdin (`mods`, `llm`, `aichat`, `smartcat`,
`shell-gpt`) or paste it into a chat web UI.

## 1. Install footprint

- `cargo install code2prompt` (recommended; pulls a fresh build).
- `brew install code2prompt` (Homebrew formula tracks releases).
- Prebuilt binaries on the GitHub Releases page for macOS / Linux /
  Windows. Single static binary, no runtime deps.
- Optional Wayland clipboard support behind the `wayland` feature
  flag: `cargo install --features wayland code2prompt`.
- Python SDK ships separately as `pip install code2prompt-rs` (Rust
  core via PyO3 bindings).
- Config file is optional and project-local: `.c2pconfig` at the
  repo root.

## 2. Repo, version, license

- Repo: <https://github.com/mufeedvh/code2prompt>
- Version checked: **v4.2.0** (released 2025-12-11).
- License: MIT. License file at the repo root: `LICENSE`.

## 3. What it actually emits

By default `code2prompt .` produces, in order:

1. A `Project Path:` header.
2. A `Source Tree:` block — an ASCII tree of all files that survived
   `.gitignore` + include/exclude globs.
3. One fenced code block per file, language tag inferred from the
   extension, file path as the header.
4. (Optional) `--diff`, `--git-log-branch <a> <b>`, or
   `--git-diff-branch <a> <b>` blocks for Git context.
5. (Optional) A token count printed to stderr (or an in-TUI counter)
   computed with the OpenAI `tiktoken` vocab.
6. The whole thing is also copied to the system clipboard unless
   `--no-clipboard` is passed; `--output-file` writes it to disk
   instead.

The output shape is fully overridable via Handlebars templates
(`-t path/to/template.hbs`). Bundled templates cover bug fixes,
documentation generation, security review, refactors. Custom
templates have access to variables like `{{absolute_code_path}}`,
`{{source_tree}}`, `{{files}}`, plus user-supplied vars at the
command line (`--var key=value`).

## 4. MCP support

Yes — the project ships an MCP server mode (separate `code2prompt-mcp`
crate in `crates/`). Started as a long-lived process, it exposes
codebase-packing tools to MCP-aware agents (`opencode`, `claude-code`,
`cline`, `OpenHands`, `crush`, `continue`) so the agent can ask for a
freshly-packed snapshot on demand instead of being handed one up
front.

This is the same niche as [`repomix`](../repomix/) `--mcp`. Pick
`code2prompt --mcp` if you are already standardising on Rust tooling
or want the Handlebars template system; pick `repomix --mcp` if you
want Tree-sitter compression or Secretlint scanning baked in.

## 5. Sub-agent model

None. It is a one-shot packer. There is no loop, no model call, no
agency.

## 6. Telemetry stance

Off. No analytics, no network calls outside of optional Git operations
on your local repo and the clipboard copy. The only egress is
whatever downstream LLM CLI you pipe the output into.

## 7. Token / context strategy

`code2prompt` itself counts tokens with `tiktoken` (OpenAI vocab) and
prints a budget figure so you can decide whether to re-run with
tighter `--include` / `--exclude` globs before paying for an API
call. It does **not** chunk or summarize; if your repo is too big,
that is your problem to solve with globs, templates, or a `symbex` /
`marker` pre-pass.

The interactive TUI (`code2prompt . --tui` or default-launch in
recent versions) lets you toggle files in a tree view and watch the
token count update live before exporting — a UX `files-to-prompt`
and `repomix` do not have.

## 8. Hot keybinds (TUI)

- `Space` — toggle file/folder selection.
- `↑ / ↓` / `j / k` — navigate the tree.
- `Enter` — expand/collapse a folder.
- `/` — filter by name.
- `c` — copy current selection to clipboard.
- `q` — quit.

(Keybinds drift across releases; `?` inside the TUI prints the
current map.)

## 9. Killer feature, weakness, when to choose

**Killer feature.** Handlebars templates as the export shape.
`files-to-prompt` is opinionated about XML/markdown; `repomix` is
opinionated about markdown/XML/plain; `code2prompt` lets you
hand-author the exact wrapper your downstream prompt expects, with
loops over files and access to Git metadata. Pair that with the
live-token-count TUI for hand-curating context, and it occupies a
slot neither `files-to-prompt` nor `repomix` does.

**Weakness.** No Tree-sitter compression and no built-in
secret-scanning — `repomix` wins on both. Token counting is
OpenAI-vocab-only, so Anthropic / Gemini / Llama numbers are
estimates. Output can balloon on monorepos if you forget to set
`--include`; the TUI helps but does not enforce a budget cap.

**When to choose.**
- You want a **single static Rust binary** with no Python runtime,
  no Node runtime, no `npx` — `code2prompt` vs `files-to-prompt`
  (Python) and `repomix` (Node).
- You need to **shape the output prompt** with a custom
  Handlebars template, not just pick markdown vs XML — `code2prompt`
  beats both neighbors here.
- You want an **interactive TUI** to curate which files go into the
  prompt with live token counts — neither neighbor offers this.
- You also want an **MCP server mode** so an agent can pack
  on-demand — `code2prompt --mcp` or `repomix --mcp`, pick by
  language preference and template needs.

**When to skip.**
- You only need Python symbol extraction → [`symbex`](../symbex/).
- You need Tree-sitter compression to fit a giant repo into a
  context window → [`repomix`](../repomix/) `--compress`.
- You want secret-scanning on the way out → [`repomix`](../repomix/)
  (Secretlint integration).
- You are packing PDFs, not source — convert with
  [`marker`](../marker/) first.

## 10. Compared to neighbors in the catalog

| Tool | Lang | Output shape control | Compression | Secret scan | TUI | MCP | Token count |
|------|------|----------------------|-------------|-------------|-----|-----|-------------|
| code2prompt | Rust | Handlebars templates | No | No | Yes (live) | Yes | tiktoken |
| [files-to-prompt](../files-to-prompt/) | Python | XML / markdown / line-numbered | No | No | No | No | None (use `ttok`) |
| [repomix](../repomix/) | Node | XML / markdown / plain | Tree-sitter | Secretlint | No | Yes | tiktoken |
| [symbex](../symbex/) | Python | Whole symbols only | Symbol-level (AST) | No | No | No | None |

Decision shortcut:

- Rust binary + custom Handlebars wrapper + interactive curation →
  `code2prompt`.
- Smallest possible Python install + glob-driven + zero ceremony →
  `files-to-prompt`.
- Whole-repo with compression + secret-scan + remote fetch →
  `repomix`.
- Specific Python functions only → `symbex`.
