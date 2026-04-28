# dprint

- **Repo:** https://github.com/dprint/dprint
- **Version:** 0.54.0 (latest stable, 2026)
- **License:** MIT ([LICENSE](https://github.com/dprint/dprint/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install dprint` · `cargo install dprint` · `npm install -g dprint` · `curl -fsSL https://dprint.dev/install.sh | sh` · prebuilt binaries on the GitHub release page · binary name is `dprint`

## What it does

`dprint` is a **pluggable, language-agnostic code formatter** with
a single CLI driver and per-language WebAssembly plugins. One
`dprint.json` at the repo root declares which plugins to load
(TypeScript, JSON, Markdown, TOML, Dockerfile, YAML, Rust via
rustfmt-shim, Python via Ruff, etc.) and which globs each plugin
owns. `dprint fmt` then formats the entire repo in parallel,
caching unchanged files by content hash. Plugins are downloaded
once into `~/.cache/dprint`, pinned by sha256 in
`dprint.json`, so every contributor and CI runner gets byte-identical
output.

The selling point versus running `prettier`+`rustfmt`+`black`+
`taplo` separately is **one command, one config, one cache, one
version-pin file**, with consistent ignore semantics
(`.dprintignore` / `excludes` in config) and a single editor
integration to wire up. The selling point versus a monolithic
formatter is that each plugin is the canonical formatter for its
language (the TypeScript plugin matches Prettier output, the JSON
plugin is the de-facto JSON formatter, etc.), shipped as Wasm so
no Node/Python/Ruby toolchain needs to be present.

## When to pick it / when not to

Reach for `dprint` in a polyglot repo — anything mixing TS, JSON,
Markdown, YAML, TOML, and a systems language — where you are
tired of orchestrating four separate formatters with four
different config files, four different ignore files, and four
different ways of being invoked from pre-commit. It is also the
right tool when CI formatting times matter: the parallel runner
plus content-hash cache makes incremental `dprint fmt` essentially
free.

Skip it for a monolingual repo where the native formatter is
already good (`cargo fmt` for a pure-Rust crate, `gofmt` for pure
Go — both are zero-config and need no wrapper). Skip it when your
team has strong opinions that diverge from the upstream plugin
defaults and you do not want to maintain plugin overrides. Skip it
in environments where downloading Wasm plugins from the public
registry on first run is blocked.

## Why it matters in an AI-native workflow

Coding agents produce edits across many file types in a single
turn — a TS source change, a YAML CI tweak, a Markdown changelog
update, a JSON config bump — and each file type wants its own
formatter to keep diffs minimal and reviewable. Running per-file
formatters is awkward from an agent loop; running one
`dprint fmt --diff <changed-files>` after every batch of edits is
trivial. The deterministic, hash-cached output also means agents
get a stable signal: if `dprint check` passes, formatting is not
the source of any subsequent test failure.

## Example invocations

```bash
# One-time: generate a starter dprint.json
dprint init

# Format everything covered by dprint.json globs
dprint fmt

# Check formatting in CI; exit non-zero if any file would change
dprint check

# Format only specific files (e.g. just-changed in a PR)
dprint fmt $(git diff --name-only origin/main)

# Show what would change without writing
dprint fmt --diff

# Add a plugin (writes the URL + sha into dprint.json)
dprint config add typescript

# Update all plugin pins to their latest versions
dprint config update

# Clear the local plugin cache
dprint clear-cache
```

## Alternatives in this catalog

- [`bacon`](../bacon/) — background runner; pairs with dprint
  (bacon to watch, dprint to format on save).
- [`just`](../just/) — task runner; the natural place to wrap
  `dprint check` as a `just fmt-check` recipe alongside lints.
- [`lefthook`](../lefthook/) — git hooks manager; the standard way
  to wire `dprint fmt --staged` into pre-commit across a team.
