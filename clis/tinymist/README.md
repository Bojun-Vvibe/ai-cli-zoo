# tinymist

> **Single-binary all-in-one Typst toolchain** — language server
> (LSP) for editor integration (VS Code, Neovim, Helix, Zed,
> Sublime, Emacs), formatter (`tinymist fmt`), document previewer
> with browser hot-reload (`tinymist preview`), package query
> (`tinymist query`), trace + profile, and a thin compile front
> end — all served from one Rust binary that vendors the upstream
> `typst` compiler crate, so opening a `.typ` file in any
> LSP-capable editor gets you completion, hover docs, jump-to-def,
> on-save format, and a live PDF/SVG preview without bolting
> together five different tools — pinned to **v0.14.16** (released
> 2026-04-05, [LICENSE](https://github.com/Myriad-Dreamin/tinymist/blob/v0.14.16/LICENSE),
> SPDX `Apache-2.0`).

Source: <https://github.com/Myriad-Dreamin/tinymist>

## TL;DR

Typst is the modern LaTeX competitor (already in this zoo as
`typst`) — fast incremental compile, sane scripting language,
PDF/SVG/PNG output. But the `typst` CLI itself is a *compiler*:
it does not speak LSP, does not format, does not preview with
hot-reload, does not introspect packages. The community filled
each of those gaps with separate projects (`typst-lsp`, then a
fork; `typst-preview`; `typstfmt`) and then **`tinymist`
absorbed and replaced all of them** — `typst-lsp` was archived
in favor of `tinymist`, `typst-preview` was merged in, and
`typstfmt` is now a tinymist subcommand. Today, if you write
Typst in any editor, `tinymist` is the integration layer.

## Install

```bash
# macOS / Linux Homebrew
brew install tinymist

# cargo (from source, latest crates.io)
cargo install --locked tinymist

# Pre-built binaries (Linux x86_64 / aarch64 / musl, macOS arm64
# / x86_64, Windows x64) for v0.14.16 live at:
#   https://github.com/Myriad-Dreamin/tinymist/releases/tag/v0.14.16

# VS Code: install the "Tinymist Typst" extension — it bundles
# the matching binary and wires up LSP + preview + format.

# Neovim (with nvim-lspconfig):
#   require('lspconfig').tinymist.setup{}

# Helix: append to ~/.config/helix/languages.toml
#   [language-server.tinymist]
#   command = "tinymist"
```

Hard prereqs: Typst documents to edit. The binary ships its own
copy of the compiler — no separate `typst` install required for
LSP or preview, though `typst-cli` is fine to keep alongside for
batch builds.

## Common invocations

```bash
# Run as an LSP server on stdio (editors invoke this for you)
tinymist lsp

# Format a Typst file in place
tinymist fmt main.typ

# Hot-reload preview in the browser (opens http://127.0.0.1:23627)
tinymist preview main.typ

# Compile (thin wrapper over the bundled compiler)
tinymist compile main.typ out.pdf

# Query a Typst document's exported metadata as JSON
tinymist query selector "<heading>" main.typ

# Trace + flamegraph a slow build
tinymist trace main.typ --output trace.json
```

## Why orthogonal to existing zoo

The zoo has the **`typst`** entry (compiler/CLI) and **`pandoc`**
(generic document converter) but **no Typst LSP / IDE
integration / preview / formatter** — those are exactly the
tools that make Typst usable in an editor instead of in a
build script. `tinymist` is the missing IDE-side half of the
Typst story; the closest siblings (`marksman` for Markdown
LSP, `taplo` for TOML LSP) cover different file types entirely.

## Caveats

- Apache-2.0; the bundled Typst compiler is also Apache-2.0.
  Both are clean for embedding.
- Version skew matters: tinymist's bundled compiler is pinned
  per release. If your project's `typst.toml` declares a
  newer compiler version than the bundled one, LSP diagnostics
  may flag features the bundled compiler does not yet know —
  upgrade tinymist or pin the project's compiler version down.
- The preview server defaults to `127.0.0.1:23627` and is
  unauthenticated; do not bind it to `0.0.0.0` on shared
  networks.
- `tinymist fmt` is *opinionated* and currently has limited
  configuration knobs — if the team has an existing custom
  Typst style guide, expect a one-time reformat diff and
  agree on the new baseline before turning on format-on-save
  in CI.
- Memory footprint: the LSP holds the entire compiled module
  graph in RAM for incremental analysis. Books / theses with
  hundreds of imported `.typ` files can sit at 300–600 MB
  resident; this is normal, not a leak.
