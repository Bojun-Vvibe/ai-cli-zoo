# superhtml

> **HTML linter, formatter, and Language Server in one Zig
> binary** — parses HTML5 against the WHATWG spec with proper
> error recovery, prints precise diagnostics with caret-and-span
> snippets, formats the document to a canonical shape, and
> speaks LSP so the same engine powers editor diagnostics on
> save in any LSP-aware editor (Neovim, VS Code, Helix, Zed).
> Pinned to **v0.6.2** (released 2025-10-13,
> [`gh api repos/kristoff-it/superhtml/releases/latest`](https://github.com/kristoff-it/superhtml/releases/latest),
> [LICENSE](https://github.com/kristoff-it/superhtml/blob/main/LICENSE),
> MIT).

Source: <https://github.com/kristoff-it/superhtml>

## TL;DR

HTML's tooling story has historically split into three islands:
**validators** (the W3C Nu HTML Checker — Java, accurate, slow,
JVM dependency), **formatters** (Prettier — ships the whole Node
toolchain just to reflow tags, and its HTML mode is permissive
about real bugs like an unclosed `<li>`), and **lint-as-you-type**
(emmet / built-in editor heuristics — fast, but they miss
structural problems like a `<div>` inside a `<p>`, mismatched
end tags, or attributes on void elements). `superhtml` collapses
the three into one ~3 MB Zig binary: `superhtml check file.html`
is the validator, `superhtml fmt file.html` is the formatter, and
`superhtml lsp` is the language server an editor connects to —
all backed by the same WHATWG-spec HTML5 parser written from
scratch in Zig with rich error recovery (a syntax error on line
12 doesn't suppress diagnostics for line 50). Diagnostics quote
the offending span with file:line:col anchors, pinpoint the
*structural* bug (e.g. "`</div>` closes a `<p>` opened on line 4
that was implicitly closed by `<div>` on line 8"), and never
require a config file to start producing useful output. Also
ships a templating dialect ("SuperHTML Template Language")
embedded in the same parser, so projects using it get the same
editor diagnostics over their templates that they get over plain
`.html`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install superhtml

# Pre-built binary from a release (Linux / macOS / Windows)
curl -L \
  https://github.com/kristoff-it/superhtml/releases/download/v0.6.2/superhtml-x86_64-linux.tar.gz \
  | tar xz && sudo mv superhtml /usr/local/bin/

# Build from source (any platform with Zig 0.13+)
git clone https://github.com/kristoff-it/superhtml && cd superhtml
zig build -Doptimize=ReleaseFast
sudo cp zig-out/bin/superhtml /usr/local/bin/

# verify
superhtml --version
```

## Representative examples

```bash
# 1. Lint one file — prints diagnostics with file:line:col + snippet
superhtml check index.html

# 2. Lint a tree, exit non-zero on any error (CI gate)
superhtml check 'src/**/*.html'

# 3. Format in place
superhtml fmt --in-place index.html

# 4. Format on stdin → stdout (editor-on-save integration)
cat index.html | superhtml fmt -

# 5. Run as an LSP server (editor configs:
#    Neovim: vim.lsp.start({cmd={'superhtml','lsp'}, ...})
#    VS Code: matklad/superhtml-vscode extension
#    Helix:   languages.toml: language-server.superhtml.command="superhtml")
superhtml lsp

# 6. Lint inside a CI workflow alongside other formatters
superhtml check 'public/**/*.html' && \
  superhtml fmt --check 'public/**/*.html'
```

## When to use vs. alternatives

- Pick **superhtml** when the project produces hand-written /
  template-rendered HTML that wants WHATWG-correct *structural*
  diagnostics (mismatched tags, void elements with children,
  block elements inside `<p>`) without a JVM, a Node toolchain,
  or a network round-trip — and the editor experience matters,
  because the same binary is the LSP.
- Pick the **W3C Nu HTML Checker (`vnu.jar`)** when canonical
  W3C conformance is the contractual deliverable (e.g. a public
  spec compliance audit) — it's the reference, but it's a Java
  app with a slow startup that doesn't fit a save-on-keystroke
  loop. Compose: superhtml in the editor + vnu in CI.
- Pick **Prettier** (with the `html` parser) when the project
  already runs Prettier across `.js` / `.ts` / `.css` / `.md`
  and HTML is one more file type in the same `prettier --write`
  command — Prettier wins the one-formatter-for-everything
  story, superhtml wins the *correctness* story (Prettier
  reflows happily through real HTML bugs).
- Pick **`htmlhint`** / **`html-validate`** when the rule
  surface itself is the deliverable (project-defined custom
  rules, allow/deny lists for tags / attrs / inline styles) —
  superhtml is opinionated about WHATWG conformance and does
  not expose a per-rule plugin API.
- Pick [`htmlq`](../htmlq/) for the orthogonal verb — htmlq
  *queries* an existing HTML document with CSS selectors;
  superhtml *validates and formats* it. Compose: superhtml fmt
  → htmlq for query.
- Pick [`marksman`](../marksman/) / [`vale`](../vale/) for the
  Markdown / prose neighbourhood — same LSP-as-linter shape,
  different format.
- Caveats: pre-1.0 (v0.x — pin the binary in CI rather than
  tracking `@latest`), the SuperHTML Template Language is the
  author's own templating dialect — projects on Jinja / Liquid
  / Handlebars / Go templates get the HTML5 lint pass over the
  *rendered* output but not over the template source itself,
  and the formatter is opinionated (no `printWidth`-style knobs
  — accept the canonical shape or skip `fmt`).
