# grcov

> **LLVM / gcov coverage aggregator** — a Rust CLI from Mozilla
> that ingests raw `.profraw` (LLVM source-based coverage),
> `.gcda` / `.gcno` (gcov), or `.info` (lcov) files produced by
> instrumented Rust / C / C++ binaries and emits a single merged
> report in any of: HTML, lcov, Cobertura XML, Coveralls JSON,
> Markdown, or text. Pinned to **v0.10.7** (released 2026-03-03,
> [LICENSE-MPL-2.0](https://github.com/mozilla/grcov/blob/master/LICENSE-MPL-2.0),
> MPL-2.0).

Source: <https://github.com/mozilla/grcov>

## TL;DR

`grcov` is the post-processor that turns raw coverage exhaust
into a usable report. The Rust toolchain (`-Cinstrument-coverage`)
and Clang (`-fprofile-instr-generate -fcoverage-mapping`) both
emit `.profraw` files at process exit; `cargo test` writes one
per test binary; CI ends up with hundreds of them per run. The
LLVM-native tools (`llvm-profdata merge` then `llvm-cov show`)
work but are awkward, slow on large fan-outs, and emit only HTML
or text. `grcov` does the merge plus rewrite in one parallelised
step, deals with source-tree mapping (mono-repos, vendored deps),
strips paths via `--ignore` / `--keep-only` globs, and emits
whichever report format your downstream tooling wants —
Codecov / Coveralls / Cobertura / lcov / HTML — from the same
input set.

It is the standard Rust CI coverage pipeline upstream
(`cargo +nightly test` → `grcov . --binary-path target/debug/
-s . -t lcov -o coverage.lcov`) and Mozilla uses it on
Firefox-scale C++ trees.

## Why it's interesting

Coverage tooling is bifurcated by language: `coverage.py` for
Python, `Istanbul` / `nyc` / `c8` for JS, `JaCoCo` for JVM,
`gcov` + `lcov` for C/C++, and a handful of competing crates for
Rust (`tarpaulin`, `cargo-llvm-cov`). `grcov` is the cross-
language LLVM-side aggregator: anything that emits LLVM source-
based coverage or gcov output goes in, one merged report comes
out. That makes it the right glue when one repository ships Rust
**and** a C extension **and** vendored C++ — single coverage
artifact, single Codecov upload.

## Install

```bash
# Cargo (any platform)
cargo install grcov

# Homebrew
brew install grcov

# Pre-built binary (CI)
curl -L https://github.com/mozilla/grcov/releases/download/v0.10.7/grcov-x86_64-unknown-linux-gnu.tar.bz2 \
  | tar xj -C /usr/local/bin

# verify
grcov --version    # grcov 0.10.7
```

## Examples

Rust source-based coverage (the modern path):

```bash
# 1. instrument
export RUSTFLAGS="-Cinstrument-coverage"
export LLVM_PROFILE_FILE="target/coverage/%p-%m.profraw"
cargo +nightly test

# 2. aggregate
grcov target/coverage \
  --binary-path target/debug \
  --source-dir . \
  --output-types html,lcov,markdown \
  --output-path target/coverage/report \
  --ignore '../*' --ignore '/*' --ignore 'target/*'

# upload coverage.lcov to Codecov
bash <(curl -s https://codecov.io/bash) -f target/coverage/report/lcov

# 3. open the HTML report
xdg-open target/coverage/report/html/index.html
```

C/C++ gcov coverage (legacy / mixed trees):

```bash
# build with gcov instrumentation
gcc --coverage -O0 -o app src/*.c
./app && ./tests/run.sh

# aggregate every .gcda + .gcno into one cobertura.xml for Jenkins
grcov . -t cobertura -o cobertura.xml --branch
```

Mono-repo with both Rust and a C extension:

```bash
# both toolchains drop their raw files into ./coverage/
grcov ./coverage \
  --binary-path ./target/debug \
  --source-dir . \
  --output-types lcov \
  --output-path merged.lcov
```

## Use when

- You ship Rust and want CI coverage that uploads cleanly to
  Codecov / Coveralls without writing a custom merger.
- Your repo mixes Rust + C/C++ (FFI extensions, vendored
  native code) and you need one merged coverage artifact.
- You're on a large C/C++ tree where `lcov`'s Perl
  implementation is the bottleneck — `grcov` is parallelised
  and typically 5-20× faster on the merge step.
- You want to swap report formats without rerunning the test
  suite (one `.profraw` set → HTML for humans, lcov for
  Codecov, Cobertura for Jenkins, all from the same input).

Skip `grcov` when you are pure Python / pure JS (use the
language-native coverage tool — `coverage.py`, `c8`), when
your team has standardised on `cargo-llvm-cov` (which wraps
this same pipeline with a friendlier Cargo subcommand and is
the right pick for Rust-only repos), or when you do not
actually need a merged report — a single `.profraw` rendered by
`llvm-cov show` is sufficient for one-binary local debugging.
