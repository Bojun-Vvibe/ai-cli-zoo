# cargo-mutants

> **Mutation-testing harness for Rust: rewrites your code with small,
> deliberately-broken edits ("mutants") — replacing `+` with `-`,
> `true` with `false`, function bodies with `Default::default()`,
> match arms with `unreachable!()` — then re-runs `cargo test` and
> reports which mutants survived. A surviving mutant is a hole in
> your test suite: the code changed and *no test failed*.** Pinned
> to **v27.0.0**, MIT
> ([LICENSE](https://github.com/sourcefrog/cargo-mutants/blob/main/LICENSE)).

- **Repo:** https://github.com/sourcefrog/cargo-mutants
- **Latest version:** v27.0.0 (released 2026-03-07)
- **License:** MIT (`LICENSE` at repo root)
- **Category:** `testing` / `mutation-testing` / `rust` / `coverage-quality`
- **Language:** Rust

## What it does

Line-coverage and branch-coverage tools (`cargo-llvm-cov`, `tarpaulin`)
answer the question "did the test suite *execute* this line?" — which
is necessary but laughably insufficient. A line can be executed by a
test that asserts nothing, and your "92% covered" project still ships
silent regressions. `cargo-mutants` answers a much sharper question:
"if I deliberately *break* this line, will any test notice?" It
parses your crate with `syn`, walks the AST, and at every interesting
spot emits a small set of plausible mutations — replace a binary
operator, flip a boolean literal, return `Default::default()` from a
function, replace a `match` arm body with `unreachable!()`, drop a
`?` operator, etc. For each mutant it writes a temporary copy of the
source tree to `target/mutants.out/`, runs `cargo build` (mutants
that fail to compile are discarded as uninteresting — they don't
represent realistic regressions), then runs `cargo test`. If the test
suite still passes with the broken code, the mutant *survived*: that
is a real test-suite gap with file, line, and the exact diff that
went undetected. The output is a deterministic list of survivors, not
a heatmap, so it goes straight into a CI gate or an issue tracker.

## Install

```bash
# All platforms (recommended — works wherever cargo does)
cargo install --locked cargo-mutants

# macOS via Homebrew
brew install cargo-mutants

# Or via cargo-binstall for a prebuilt binary
cargo binstall cargo-mutants
```

## Examples

```bash
# Run the full mutation-testing pass on the current crate
cargo mutants

# Restrict to one file (fast iteration while writing tests)
cargo mutants --file src/parser.rs

# Skip mutants in code matching a regex (e.g. logging, panic paths)
cargo mutants --exclude-re 'log::|panic!'

# CI mode: fail the build if any mutant survives, and emit JSON
cargo mutants --json --in-place=false --error-exit-code 1

# List mutants that *would* be generated, without running tests
cargo mutants --list
```

## Why it matters in an AI-native workflow

When an agent (Claude Code, Aider, Codex, Cline — all already in
this zoo) writes code *and* writes the tests for that code in the
same session, line coverage is meaningless: the agent will
cheerfully produce a test that calls the function and asserts
nothing, hitting 100% coverage on garbage. `cargo-mutants` is the
only widely-available Rust tool that catches this failure mode
mechanically — if the agent's test doesn't actually constrain
behaviour, a mutant survives and the CI gate flags it with a
specific file:line and a specific diff the test missed. Feed that
diff back to the agent ("this mutation went undetected, write a
test that fails on it") and the next iteration produces a real
test. Pairs with [`cargo-nextest`](../cargo-nextest/) (the parallel
test runner — mutation testing fans out massively, so a 3× faster
test run is a 3× faster mutation pass), [`bacon`](../bacon/)
(file-watcher to re-run mutants on the changed file only), and
[`cargo-deny`](../cargo-deny/) (supply-chain gate; mutants is the
behavioural gate on the same CI). Orthogonal to
[`cargo-machete`](../cargo-machete/) (which finds *unused* deps,
not untested code) and to coverage tools like `cargo-llvm-cov`
(which measures execution, not assertion strength).
