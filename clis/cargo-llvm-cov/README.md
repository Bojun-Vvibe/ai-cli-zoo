# cargo-llvm-cov

> **Source-based code coverage for Rust via LLVM's instrumentation
> profile, run as a `cargo` subcommand: `cargo llvm-cov` builds with
> `-C instrument-coverage`, runs the test suite, and emits HTML /
> LCOV / Cobertura / JSON / text reports without GCC `gcov` and
> without the legacy `grcov` toolchain dance.** Pinned to **v0.8.5**,
> dual MIT / Apache-2.0
> ([LICENSE-APACHE](https://github.com/taiki-e/cargo-llvm-cov/blob/main/LICENSE-APACHE),
> [LICENSE-MIT](https://github.com/taiki-e/cargo-llvm-cov/blob/main/LICENSE-MIT)).

- **Repo:** https://github.com/taiki-e/cargo-llvm-cov
- **Latest version:** v0.8.5 (2026-03-20)
- **License:** MIT OR Apache-2.0 (`LICENSE-MIT`, `LICENSE-APACHE` at repo root)
- **Category:** `rust` / `code-coverage` / `testing` / `ci-gate`
- **Language:** Rust

## What it does

`cargo-llvm-cov` makes Rust code coverage as boring as `cargo test`.
Under the hood it sets `RUSTFLAGS="-C instrument-coverage"` and
`LLVM_PROFILE_FILE` for the test invocation, runs `cargo test`
(or `cargo nextest`, or any `--` passthrough binary), collects the
resulting `.profraw` files, calls the `llvm-profdata` and
`llvm-cov` binaries that ship with the *exact* `rustc` toolchain in
use to merge + render them, and writes a report. The "exact
toolchain" point is the load-bearing one: the profile-data format
changes between LLVM versions, so a coverage tool that calls a
system `llvm-cov` randomly mismatches whatever LLVM rustc was
built against and silently produces wrong line numbers — `cargo
llvm-cov` instead resolves the LLVM tools through `rustup
component add llvm-tools-preview`, guaranteeing a match. Output
formats are the union of every CI sink that matters: `--html` (a
self-contained directory you `aws s3 sync` for the build artefact),
`--lcov --output-path lcov.info` (Codecov / Coveralls / Sonar
upload), `--cobertura` (Jenkins / GitLab CI native), `--json`
(custom thresholds), and the default text summary for the
PR-comment use case. Workspace-wide coverage (`--workspace`),
per-crate exclusion (`--ignore-filename-regex 'tests/'`), branch
coverage (`--branch`), MC/DC (`--mcdc` for the safety-critical
crowd), and `--fail-under-lines 80` for the CI gate are all flags,
not separate tools.

## Install

```bash
# All platforms — cargo install (the canonical path)
cargo install cargo-llvm-cov --locked

# macOS / Linux via Homebrew
brew install cargo-llvm-cov

# Required toolchain component (one-time)
rustup component add llvm-tools-preview
```

## Examples

```bash
# Run the test suite + emit a text summary
cargo llvm-cov

# Self-contained HTML report under target/llvm-cov/html/
cargo llvm-cov --html --open

# LCOV file for Codecov / Coveralls upload
cargo llvm-cov --lcov --output-path lcov.info

# Whole workspace, no doctests, exclude generated code
cargo llvm-cov --workspace --no-fail-fast \
  --ignore-filename-regex '(target/|tests/fixtures/)'

# CI gate: fail if line coverage drops below 80%
cargo llvm-cov --workspace --fail-under-lines 80

# Use cargo-nextest as the runner (faster + better isolation)
cargo llvm-cov nextest

# Branch coverage (recent rustc nightly)
cargo llvm-cov --branch --html

# Sequence: clean profraw, run, report (separate steps for caching)
cargo llvm-cov clean --workspace
cargo llvm-cov --no-report
cargo llvm-cov report --html
```

## Why it matters in an AI-native workflow

LLM-generated Rust code reliably ships *some* tests and reliably
under-tests the error paths the model did not think to enumerate
(the `Err(_)` arm of every `?`, the timeout branch, the
`Vec::is_empty()` early-return). A line-coverage gate is the
cheapest mechanical check that catches "the agent wrote 200 lines
of new code and 12 lines of new test." `cargo-llvm-cov` is the
right tool to run that gate: it is a single `cargo install`, it
works against the toolchain the agent and the reviewer already
share, and `--fail-under-lines` is a one-flag CI assertion that
needs no Rust-specific reviewer expertise to enforce.
Pairs naturally with [`cargo-nextest`](../cargo-nextest/)
(`cargo llvm-cov nextest` runs nextest's faster + per-test process
isolation harness while still collecting profile data — strict
upgrade for any project that has both), and with
[`cargo-mutants`](../cargo-mutants/) on the orthogonal axis:
line coverage tells you *whether the line ran*, mutation testing
tells you *whether running the line mattered to any assertion*.
A serious Rust CI runs all three: nextest as the runner,
`cargo-llvm-cov` as the coverage gate on every PR,
`cargo-mutants` as the deeper nightly job that catches the
assertion-free tests `cargo-llvm-cov` would happily count as
covered. Distinct from
[`cargo-tarpaulin`](https://github.com/xd009642/tarpaulin) (the
older Rust coverage tool, which uses `ptrace` instrumentation on
Linux and tends to drift in accuracy on macOS / Windows;
`cargo-llvm-cov` is platform-uniform because it uses the same
LLVM instrumentation Clang/Swift use) and from `grcov` (the older
Mozilla-derived path that is now mostly subsumed — `cargo-llvm-cov`
is what Rust-project maintainers themselves recommend for new
setups).
