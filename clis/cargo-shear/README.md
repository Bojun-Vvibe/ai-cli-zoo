# cargo-shear

> **Detect and remove unused dependencies from Rust crates and
> workspaces** — a single-binary `cargo` subcommand that walks every
> crate in a workspace, parses its `[dependencies]` / `[dev-dependencies]`
> / `[build-dependencies]` from `Cargo.toml`, scans the actual `use` /
> `extern crate` / macro-call graph in `src/` with `syn`, and reports
> (or with `--fix` deletes) every entry that is declared but never
> imported. Pinned to **v1.11.2** (released 2026-03-15,
> [`gh api repos/Boshen/cargo-shear/releases/latest`](https://github.com/Boshen/cargo-shear/releases/latest),
> [LICENSE](https://github.com/Boshen/cargo-shear/blob/main/LICENSE),
> MIT).

Source: <https://github.com/Boshen/cargo-shear>

## TL;DR

A long-lived Rust workspace accumulates dead `Cargo.toml` entries
the same way a long-lived JS project accumulates dead `package.json`
entries — somebody added `regex` for a one-off, deleted the call
site, and the dependency stayed forever, dragging compile time and
`cargo audit` surface with it. `cargo-shear` is the targeted answer:
one `cargo shear` invocation in repo root prints the unused entries
per crate, exit-non-zero so CI can gate, and `cargo shear --fix`
mutates the `Cargo.toml` files in place. It also detects
*workspace*-level cruft (entries in `[workspace.dependencies]` that
no member crate uses) and per-crate `default-features = false` flags
that no longer matter. The author (Boshen, also the lead of `oxc` /
`oxlint`) ships it as a pure-Rust dependency-graph walker — no
network, no Cargo registry calls, no `cargo metadata` round-trip on
the hot path — so on a 50-crate workspace it finishes in single-digit
seconds where the historical alternative (`cargo-udeps`) takes
minutes because it forces a full `cargo check` first.

## Install

```bash
# Cargo (any platform with a Rust toolchain)
cargo install cargo-shear --locked

# Homebrew (macOS / Linux)
brew install cargo-shear

# Pre-built binary from a release (Linux / macOS / Windows)
curl -L \
  https://github.com/Boshen/cargo-shear/releases/download/v1.11.2/cargo-shear-x86_64-unknown-linux-gnu.tar.gz \
  | tar xz && sudo mv cargo-shear /usr/local/bin/

# verify
cargo shear --version    # cargo-shear 1.11.2
```

## Representative examples

```bash
# 1. Scan the current workspace, print unused deps per crate
cargo shear

# 2. CI gate: exit non-zero if any unused dep exists
cargo shear || { echo "unused deps detected"; exit 1; }

# 3. Auto-remove all unused entries from every Cargo.toml in the workspace
cargo shear --fix

# 4. Scan a specific crate path (not the cwd workspace)
cargo shear --path crates/parser/

# 5. Ignore false positives (macros / build.rs uses) via the
#    [package.metadata.cargo-shear] block in Cargo.toml:
#       [package.metadata.cargo-shear]
#       ignored = ["proc-macro2"]
```

## When to use vs. alternatives

- Pick **cargo-shear** when the workspace is large, you want a
  fast static-analysis pass that runs in CI on every PR, and
  `--fix` to mutate `Cargo.toml` in place is the actual deliverable.
- Pick **cargo-udeps** when you specifically need *Nightly* `cargo`
  semantic-level unused-dep detection (catches deps used only via
  `cfg`-gated targets that cargo-shear's syntactic scan misses), and
  you can afford a full `cargo check` build first.
- Pick [`cargo-machete`](../cargo-machete/) when you want a similar
  fast static walker with a slightly different ignore-rules surface
  — both tools are fine; cargo-shear has tighter workspace-deps
  handling and `--fix` ergonomics, cargo-machete has a longer
  install base and a Mozilla-shaped rule for known false-positives.
- Pick [`cargo-deny`](../cargo-deny/) for the orthogonal concern:
  *license / advisory / source-allowlist* gating on the deps you
  *do* keep — cargo-shear deletes the dead ones, cargo-deny polices
  the live ones.
- Skip altogether if the workspace is one binary crate with five
  deps — `cargo tree` plus eyeballs is faster than installing a
  tool.
