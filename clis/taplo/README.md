# taplo

> **A single Rust binary that is the missing TOML toolchain — formatter,
> linter, JSON-Schema-aware validator, and language server — driven from
> one `taplo.toml` config and shipping the same logic the VS Code
> "Even Better TOML" extension already uses.** Pinned to **v0.10.0**,
> MIT ([LICENSE](https://github.com/tamasfe/taplo/blob/master/LICENSE)).

- **Repo:** https://github.com/tamasfe/taplo
- **Latest version:** 0.10.0
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `formatter` / `linter` / `language-server`
- **Language:** Rust
- **Install:** `brew install taplo` · `cargo install taplo-cli --locked`
  · `npm install -g @taplo/cli` · prebuilt binaries on the GitHub
  release page · binary name is `taplo`

## What it does

`taplo` treats TOML as a first-class language with a real toolchain
instead of "the file format `Cargo.toml` happens to use". The single
binary exposes four orthogonal subcommands that share the same parser:

1. **`taplo format`** — opinionated formatter with knobs for indent,
   alignment of equal signs, array-of-tables spacing, key reordering,
   and trailing newlines. Runs in-place by default; `--check` is the
   CI-safe non-modifying variant. Output is byte-stable so it is safe
   to wire into pre-commit and required-status checks.
2. **`taplo lint`** — pure semantic checks (duplicate keys, invalid
   datetime literals, mixed-type arrays under strict mode, dotted-key
   collisions) that the spec allows but humans rarely want.
3. **`taplo check`** — schema validation against a JSON Schema. The
   `[[rule]]` block in `taplo.toml` maps glob patterns to a schema URL
   or local path; the embedded schema store ships with up-to-date
   schemas for `Cargo.toml`, `pyproject.toml`, `rustfmt.toml`,
   `dprint.json`, `.cargo/config.toml`, `rust-toolchain.toml`,
   GitHub Actions reusable workflows in TOML form, and dozens more.
   Custom schemas drop in by URL.
4. **`taplo lsp stdio`** — a Language Server Protocol implementation
   that powers the "Even Better TOML" VS Code extension and any other
   LSP-aware editor (Helix, Neovim via `nvim-lspconfig`, Zed, Sublime
   via `LSP`). Same formatter, same linter, same schema engine — so
   the editor experience and the CI gate cannot drift.

The Rust crate `taplo` is reusable as a library, which is how
`dprint-plugin-toml` and several Rust IDEs ship the same engine
without re-implementing TOML parsing.

## Why included

Most teams treat TOML as plain text — `Cargo.toml`, `pyproject.toml`,
`rust-toolchain.toml`, `.cargo/config.toml`, `pre-commit-hooks.toml`,
agent config files, MCP server manifests, model registry pins — and
let it drift into a slow accumulation of duplicate keys, inconsistent
alignment, schema-invalid blocks that only fail at runtime, and the
classic `[dependencies]` block that is half alphabetised and half not.
`taplo` collapses that into one binary and one config: format on save
in the editor, `taplo format --check` in CI, `taplo check` for
schema-aware validation against the schema store. The combination
matters because the alternatives are partial — `dprint` formats but
does not validate against schemas, `cargo fmt` only knows about Rust
source, IDE plugins format but do not run in CI. `taplo` is the only
tool that ships the formatter, the schema validator, and the LSP from
the same parser, which means the editor diagnostic and the CI failure
are guaranteed to match.

In an AI-native workflow this matters more, not less: agents generate
TOML the same way they generate code — fluently, frequently, and with
occasional duplicate-key or wrong-type errors that are invisible
until a `cargo build` or `pip install` blows up far from the edit
site. A pre-commit `taplo format && taplo check` gate catches both
classes at write time. Pairs naturally with [`dprint`](../dprint/)
(unified multi-language formatter that uses `dprint-plugin-toml`
under the hood — both pinned to taplo's parser) and with
[`pkl`](../pkl/) (when you outgrow TOML's expressiveness for
config-as-code).

## Tradeoffs

- The opinionated formatter has fewer knobs than `dprint-plugin-toml`'s
  full surface; if you want column-perfect alignment of complex inline
  tables, the defaults will sometimes re-flow them.
- Schema-aware checks only catch what the schema declares; an
  out-of-date schema in the embedded store will let invalid keys
  through silently. Pin schema URLs in `taplo.toml` for reproducibility.
- The LSP is fast but assumes a single TOML file at a time; cross-file
  references (e.g. workspace-inheritance in `Cargo.toml`) are not
  resolved, so "go to definition" on `workspace = true` does not jump.
- MIT-only (not the more common Rust dual MIT/Apache); a non-issue for
  most consumers but worth noting if your distribution policy
  specifically requires Apache-2.0.
