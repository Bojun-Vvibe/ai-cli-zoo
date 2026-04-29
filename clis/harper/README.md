# harper

> **Offline, privacy-first grammar and style checker** —
> a Rust-powered linter for English prose that runs entirely
> on your machine with no cloud round trip. Pinned to
> **v2.1.0** (commit
> `cc965dba54dfb2ed272f9c271bc108cddee98e64`,
> [LICENSE](https://github.com/Automattic/harper/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/Automattic/harper>

## TL;DR

`harper` is what you reach for when you want grammarly-style
prose feedback but cannot (or will not) ship your draft to
a SaaS endpoint. The core is a Rust library; ships are a CLI
(`harper-cli`), an LSP server (`harper-ls`) that any editor
with LSP support can talk to, plus official Obsidian, VS
Code, Zed, and browser extensions. Rule set covers spelling,
grammar agreement, repeated words, capitalization, common
confusables (`their`/`there`/`they're`), spacing, redundant
phrases, and a growing style layer. No model download, no
account, no telemetry — just a binary that lints prose.

## Install

```bash
# Homebrew (macOS / Linux)
brew install harper

# Cargo (CLI + LSP)
cargo install --locked harper-cli
cargo install --locked harper-ls

# Pre-built binary from a release
curl -LO "https://github.com/Automattic/harper/releases/download/v2.1.0/harper-cli-aarch64-apple-darwin.tar.gz"
tar xf harper-cli-aarch64-apple-darwin.tar.gz
sudo install harper-cli /usr/local/bin/

# verify
harper-cli --version    # 2.1.0
harper-ls --version     # 2.1.0
```

For editor integration, point your LSP client at `harper-ls`
and configure it for `markdown`, `text`, and the source-code
languages whose comments / strings you want linted.

## License

Apache-2.0 — see
[LICENSE](https://github.com/Automattic/harper/blob/master/LICENSE).
Permissive with patent grant; safe to bundle in commercial
products and proprietary editors. Attribution required in
distribution but not in invocation output.

## One Concrete Example

```bash
# 1. lint a single Markdown file from the CLI
harper-cli lint README.md
# README.md:14:5  Use the proper version of the noun "Github" → "GitHub"
# README.md:42:1  Repeated word: "the the"

# 2. lint stdin (handy for pipelines)
echo "this is is a test" | harper-cli lint --stdin
# stdin:1:9  Repeated word: "is is"

# 3. lint every Markdown file in a docs tree
fd -e md . docs/ -x harper-cli lint {} \;

# 4. emit JSON for CI consumption
harper-cli lint --format json README.md \
  | jq '.[] | {line, col: .span.start, msg: .message}'

# 5. start the LSP server (editors auto-spawn this; useful
# when wiring it manually)
harper-ls --stdio

# 6. lint code comments and string literals (Rust example)
harper-cli lint src/main.rs

# 7. as a pre-commit hook to catch typos in commit messages
# .git/hooks/commit-msg:
#   harper-cli lint --stdin < "$1" || exit 1

# 8. dump every active rule
harper-cli rules
```

## Niche It Fills

**Local LSP-shaped grammar checker for editors and CI.**
Every other prose checker either calls home (Grammarly,
LanguageTool's hosted endpoint, GPT-based linters) or is
trapped inside one editor (Word, Pages). `harper` is a
single binary with a stdin lint mode for pipelines and an
LSP mode for editors — the same engine answers `git
pre-commit` and your inline squiggle in VS Code.

## Why use it

Three things `harper` does that the alternatives do not:

1. **Fully offline by design.** No model weights to fetch,
   no API key to register, no network egress at any point.
   Drop the binary on an air-gapped box; it works.
2. **LSP-first.** `harper-ls` speaks Language Server
   Protocol, so every editor with LSP support — Neovim, VS
   Code, Zed, Helix, Emacs (via lsp-mode), Sublime — gets
   the same feedback with the same fix actions. No editor
   plugin bit-rot.
3. **Lints code-comment prose, not just `.md`.** Point it
   at `src/`; it parses the source language, extracts
   comments and string literals, and lints those for
   grammar — catching the typos in your docstrings without
   complaining about `let x = 42;`.

## Vs Already Cataloged

- **Vs [`typos`](../typos/):** complementary — `typos`
  finds *misspellings* (one-word, dictionary-based) at
  enormous speed across an entire repo; `harper` finds
  *grammar and style issues* in prose. Run `typos` in CI
  for the spelling pass; run `harper` on `docs/` and
  comments for the grammar pass. They overlap on simple
  typos but disagree on style.
- **Vs [`markdownlint-cli2`](../markdownlint-cli2/):**
  orthogonal — markdownlint enforces *Markdown structure*
  (heading levels, list spacing, link syntax); harper
  reads the prose *content*. Run both; they catch
  disjoint issue classes.
- **Vs LanguageTool (not cataloged):** sibling — both lint
  prose with hand-written rules. LanguageTool is JVM
  (~300 MB), runs as a server, has a bigger rule set, and
  supports many languages. Harper is a single static Rust
  binary, English-only today, much faster startup, and
  no JVM dependency. Pick LanguageTool for non-English or
  rule-set depth; pick harper for editor integration and
  CI on English prose.
- **Vs `vale` (sibling):** vale is markup-aware
  (`.md`/`.rst`/`.adoc` parser) and rule-pack-driven (you
  ship YAML rules per project / per house style); harper
  ships a single curated rule set and an LSP server. Use
  vale when you want a per-project style guide enforced;
  use harper when you want grammar-checker squiggles in
  every editor.

## Caveats

- **English only (today).** The rule set is English; other
  languages are not supported. Multi-language docs need a
  different tool (LanguageTool) or a hybrid setup.
- **No autocorrect for nuanced grammar.** Harper offers
  fixes for unambiguous issues (repeated words, spacing,
  obvious typos) but leaves stylistic suggestions as
  diagnostics — you accept or reject manually. That is by
  design; do not expect Grammarly-style rewrites.
- **Rule customization is global, not per-project.** You
  can disable rules and add words to a personal
  dictionary, but a sharable per-repo rule pack (vale-style
  YAML) is not the harper model. Ship a `.harper.json` if
  the project needs it, but expect less granularity than
  vale.
- **Code-comment linting parses common languages, not
  every dialect.** Modern Rust / Python / JavaScript /
  TypeScript / Markdown / Go are well-supported; obscure
  DSLs may not have a parser yet — file an issue or fall
  back to `--stdin` with the prose pre-extracted.
- **The "Automattic" owner on GitHub may surprise license
  scanners** that expect an "Automattic, Inc." line. The
  Apache-2.0 LICENSE file is canonical; cite it directly
  if asked.
