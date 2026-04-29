# cargo-machete

> **Find unused Rust dependencies in your `Cargo.toml`** —
> a fast, false-positive-tolerant linter that scans the
> source tree of every crate in a workspace and reports
> dependencies declared in `Cargo.toml` that are not actually
> referenced in `src/`, `tests/`, `benches/`, or `examples/`.
> Pinned to **v0.9.2**
> ([LICENSE.md](https://github.com/bnjbvr/cargo-machete/blob/main/LICENSE.md),
> MIT).

Source: <https://github.com/bnjbvr/cargo-machete>

## TL;DR

`cargo-machete` is the Rust answer to `depcheck` (Node) /
`unimport` (Python) / `goimports` (Go): a tool that walks
your `Cargo.toml`s and flags entries in `[dependencies]`,
`[dev-dependencies]`, and `[build-dependencies]` that no
file in the corresponding crate actually uses. It is
intentionally syntax-driven rather than semantic — it greps
for `use foo::`, `foo::bar`, `extern crate foo`, and
`#[macro_use] extern crate foo` patterns across `.rs` files
rather than running a full compile-time analysis — which
makes it fast (sub-second on most workspaces, a few seconds
even on large ones like `wgpu` or `bevy`) and language-server-
free, at the cost of needing an explicit ignore list for
crates that *are* used in ways the grep can't see (proc-macro
crates re-exported from another crate, crates whose only
use is via `extern crate` in a build script, crates pulled
in only as features). The ignore list lives in
`Cargo.toml` under `[package.metadata.cargo-machete]` so it
travels with the crate. There is also a `--fix` flag that
rewrites the manifests to remove the unused entries (review
the diff before committing). Bundled as a `cargo` subcommand,
so once installed it runs as `cargo machete` from any
workspace root.

## When to choose

- **You're cleaning up a long-lived Rust project** that has
  accumulated dependencies via "let me try this crate"
  experiments that were never reverted. Each unused
  dependency adds compile time, target/ size, supply-chain
  surface, and `cargo audit` noise.
- **You're enforcing a "no dead deps" rule in CI** —
  `cargo machete` exits non-zero when it finds something,
  so dropping it into a GitHub Actions / GitLab CI step is
  a one-liner that prevents new dead deps from landing.
- **You're shrinking compile time on a workspace** — every
  removed dependency is one less crate to download, parse,
  and link. On large workspaces (50+ crates), the cumulative
  savings of dropping 5-10 unused deps per crate is
  measurable in clean-build time.
- **You're auditing a Cargo workspace inherited from
  another team** — the tool gives you a fast, mechanical
  inventory of what's *declared* but not *referenced*,
  which is a starting point for understanding what the
  previous maintainer was actually doing.

## When NOT to choose

- **You rely heavily on conditional compilation** — `#[cfg(...)]`
  feature gates can hide legitimate uses behind feature flags
  the scanner doesn't enable. `cargo machete` will produce
  false positives unless you list those crates in
  `[package.metadata.cargo-machete] ignored = [...]`.
- **Your code re-exports proc-macro crates** — if `crate-A`
  declares `serde_derive` but only consumes it via a
  re-export from `crate-B`, the scanner sees no `use` of
  `serde_derive` in `crate-A` and flags it. Use the
  ignore list.
- **You want a semantic, compiler-grade analysis** — for a
  ground-truth answer use `cargo udeps`
  ([upstream](https://github.com/est31/cargo-udeps)), which
  hooks into the compiler via nightly Rust. `cargo udeps`
  is more accurate but slower and requires a nightly
  toolchain; `cargo machete` is faster and stable-only.
- **You want a license / supply-chain audit** — that's
  [`cargo-deny`](https://github.com/EmbarkStudios/cargo-deny)
  and [`cargo-audit`](https://github.com/RustSec/rustsec).
  `cargo-machete` only answers "is this dep used".

## Install

```sh
# Cargo (any platform with Rust 1.74+)
cargo install --locked cargo-machete --version 0.9.2

# Arch Linux (AUR)
yay -S cargo-machete

# Nix
nix-shell -p cargo-machete

# verify
cargo machete --version    # cargo-machete 0.9.2

# run against the current workspace
cargo machete

# auto-fix (rewrites Cargo.toml; review the diff)
cargo machete --fix

# CI mode (exit code only, no interactive prompts)
cargo machete --skip-target-dir
```

## What it does

- Walks every crate in the current workspace (or a passed
  path) and reads each `Cargo.toml`.
- For each declared dependency in `[dependencies]`,
  `[dev-dependencies]`, `[build-dependencies]`, scans the
  corresponding source roots (`src/`, `tests/`, `benches/`,
  `examples/`, `build.rs`) for `use foo::`, `foo::`,
  `extern crate foo`, `#[macro_use] extern crate foo`.
- Reports anything declared but not found.
- Honours `[package.metadata.cargo-machete] ignored = [...]`
  per-crate for legitimate false positives.
- `--fix` rewrites the manifests in place to drop the
  unused entries; `--with-metadata` includes path, git, and
  registry deps.
- Returns non-zero exit code on findings → CI-friendly.

## pew-related use cases

- **Quarterly dep cleanup** on a long-lived service: run
  `cargo machete --fix`, review the diff, run the test
  suite, commit. Removes 3-10 deps in 60 seconds on
  most projects.
- **Pre-merge CI gate** on a Rust monorepo: a
  `cargo machete --skip-target-dir` step prevents PRs from
  shipping new dead deps.
- **Crate-extraction refactor**: when splitting one large
  crate into several smaller ones, run `cargo machete`
  after each extraction step — anything it flags is
  probably a dep that moved to a sibling crate but didn't
  get removed from the parent.
- **Audit before publishing a `[lib]` crate to crates.io**:
  unused deps are visible in your crate's published
  manifest and propagate as transitive overhead to every
  downstream user. Cleaning them up before release is
  cheap good citizenship.

## Comparable tools

- **`cargo udeps`** ([upstream](https://github.com/est31/cargo-udeps))
  — the semantic alternative. Hooks into the compiler via
  nightly Rust to get a ground-truth answer about which
  deps were actually compiled-in. More accurate, slower,
  needs nightly. Use `udeps` when `machete`'s false-positive
  rate becomes annoying; use `machete` for fast,
  stable-toolchain CI.
- **`cargo-shear`** ([upstream](https://github.com/Boshen/cargo-shear))
  — newer competitor with similar scope; written by the
  Oxc / Rolldown maintainer. Worth comparing if `machete`
  produces too many false positives on your code.
- **`cargo-deny`** ([upstream](https://github.com/EmbarkStudios/cargo-deny))
  — different problem (license / advisory / source
  policy). Complementary, not overlapping.
- **`cargo-bloat`** ([upstream](https://github.com/RazrFalcon/cargo-bloat))
  — answers "which deps contribute the most *bytes* to my
  binary" rather than "which are unused". Use after
  `machete` when you've removed the dead deps and want to
  shrink the live ones.
