# kondo

> **A project-aware "rm -rf node_modules / target / build" for ~24
> ecosystems** — one Rust binary that walks a tree, recognises each
> project's build cache (Rust `target/`, Node `node_modules/`,
> Python `__pycache__/`, Java `target/`, Zig `zig-cache/`,
> Terraform `.terraform/`, React Native `ios/build/`, …), and
> reclaims the disk without touching source. Pinned to **v0.9**
> ([LICENSE](https://github.com/tbillington/kondo/blob/master/LICENSE),
> MIT).

Source: <https://github.com/tbillington/kondo>

## Category

Filesystem / dev-disk reclaimer. Sits between manual
`find . -name node_modules -exec rm -rf {} +` one-liners and
heavyweight cleanup GUIs. Read-only by default; deletion is opt-in
per project via interactive prompt or `--all`.

## What it does

`kondo` recursively scans a directory, classifies every project
it finds into one of ~24 known types (Cargo, npm/yarn/pnpm,
Maven, Gradle, Stack/Cabal, Pixi, Turborepo, Zig, Terraform,
React Native, Unity, Unreal, Godot, Jupyter, …), and reports
how many bytes each project's reclaimable artifacts are
holding. You then approve cleanups one project at a time
(default), or pass `--all`/`--dry-run`/`--older 30d` for batch
scripting. Deletion paths are hard-coded per project type, so
it never nukes source files or VCS state.

## Why it matters

The "I'm out of disk and 80 % of it is build caches I'll never
re-use" problem is universal but tedious to script safely:
`find` matches too broadly, IDE cleaners are per-language,
and full nukes risk hitting `.git`. `kondo` encodes "what's
safely deletable per ecosystem" once and reuses it everywhere.
v0.9 added `--dry-run` (preview reclaim), `single-key` mode
(one keystroke per project — perfect for fast triage), and
support for Pixi, Turborepo, Haskell Cabal, React Native, and
Terraform — covering the common modern polyglot monorepo.

## Install

```bash
# Homebrew
brew install kondo

# Cargo
cargo install kondo --locked

# Pre-built binaries (v0.9, sha256 in release notes)
# macOS arm64:
curl -L https://github.com/tbillington/kondo/releases/download/v0.9/kondo-aarch64-apple-darwin.tar.gz | tar xz
# Linux x86_64 musl (static):
curl -L https://github.com/tbillington/kondo/releases/download/v0.9/kondo-x86_64-unknown-linux-musl.tar.gz | tar xz

# Verify
kondo --version    # kondo 0.9
```

## License

MIT — see
[LICENSE](https://github.com/tbillington/kondo/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# walk ~/code, list reclaimable per project, prompt y/n each
kondo ~/code

# preview only — never delete, just show totals
kondo --dry-run ~/code | sort -k2 -h

# fast triage: one keypress per project (y/n/q/?), no enter needed
kondo ~/code  # then use the single-key mode hotkeys

# nuke every project older than 30 days, no prompts (CI / cron)
kondo --all --older 30d ~/code

# pipe-friendly: just the path + bytes, for jq / sort
kondo --dry-run ~/code -- --json | jq '.[] | select(.size > 1e9)'
```

## Niche It Fills

Generic "find big dirs" tools (`du`, `ncdu`, `dust`, `erdtree`)
tell you *what* is large. Per-language cleaners (`cargo clean`,
`npm prune`, `mvn clean`) only know one ecosystem. `kondo`
is the only one that is *both* polyglot and project-aware:
you point it at a multi-repo workspace, and it gives you a
ranked, per-project, safe-to-delete list — then deletes
exactly the right paths for each project type. The killer
combo in practice is `kondo --older 30d --all ~/code` in a
cron — it keeps a dev laptop's reclaimable build noise
permanently bounded without any per-repo config.
