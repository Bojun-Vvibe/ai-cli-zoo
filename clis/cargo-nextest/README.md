# cargo-nextest

> **A next-generation test runner for Rust.** A drop-in `cargo
> test` replacement that runs each test in its own process,
> parallelises across cores by default, prints results in a
> compact live-updating tree, supports retries / partitions /
> per-test timeouts / per-test resource limits, and emits
> JUnit-shaped XML so CI surfaces native flake reports. Pinned
> to **cargo-nextest-0.9.133**
> ([LICENSE-APACHE](https://github.com/nextest-rs/nextest/blob/main/LICENSE-APACHE)
> / [LICENSE-MIT](https://github.com/nextest-rs/nextest/blob/main/LICENSE-MIT),
> dual Apache-2.0 / MIT).

Source: <https://github.com/nextest-rs/nextest>

## TL;DR

`cargo nextest run` is what `cargo test` should have been on a
modern multi-core machine. Each test binary is invoked once
per test (process-per-test), so a panic / abort / `std::process::exit`
in one test cannot poison the run; tests are scheduled across
all cores; output is buffered per-test and streamed only when
the test finishes (or fails), so logs are readable instead of
interleaved; failures are re-printed at the end as a summary.
On top of that: `--retries 2` for flaky tests, `--partition
hash:1/4` for splitting a suite across CI shards, per-test
timeouts (`slow-timeout` / `leak-timeout`), filterset DSL
(`-E 'test(my_module::)'`), and JUnit XML output that GitHub
Actions / Buildkite / TeamCity all parse. For an LLM-CLI
workflow, it is the test runner that gives the agent a
machine-readable, deterministic signal of "did the change
break anything?" without parsing `cargo test`'s console
spew.

## Install

```bash
# Cargo (from crates.io, source build)
cargo install cargo-nextest --locked

# Cargo (prebuilt binary via cargo-binstall, much faster)
cargo binstall cargo-nextest

# Homebrew
brew install cargo-nextest

# from the official install script (prebuilt tarball)
curl -LsSf https://get.nexte.st/latest/mac | tar zxf - -C ~/.cargo/bin
# linux: https://get.nexte.st/latest/linux
# windows: https://get.nexte.st/latest/windows-tar

# Nix
nix-env -iA nixpkgs.cargo-nextest

# verify
cargo nextest --version    # cargo-nextest 0.9.133
```

Zero config to start; `cargo nextest run` works in any
workspace that `cargo test` works in. Project-level config
goes in `.config/nextest.toml` at the workspace root and is
where you set per-test timeouts, default retries, profile
overrides for CI, and filtersets.

## License

Dual Apache-2.0 / MIT — see
[LICENSE-APACHE](https://github.com/nextest-rs/nextest/blob/main/LICENSE-APACHE)
and
[LICENSE-MIT](https://github.com/nextest-rs/nextest/blob/main/LICENSE-MIT).
Permissive, pick whichever attribution flow your project uses.

## One Concrete Example

```bash
# 1. baseline run, replacing `cargo test`
cargo nextest run

# 2. just one crate's tests, with output streaming
cargo nextest run -p my-crate --no-capture

# 3. filterset DSL: every test in module `db::` not marked `#[ignore]`
cargo nextest run -E 'test(db::) and not test(ignored)'

# 4. retry flaky tests up to twice; emit JUnit XML for CI
cargo nextest run --retries 2 \
  --profile ci   # picks the [profile.ci] block in .config/nextest.toml

# 5. CI sharding — same command on each shard, different fraction
cargo nextest run --partition hash:1/4    # shard 1 of 4
cargo nextest run --partition hash:2/4    # shard 2 of 4

# 6. list tests that *would* run, machine-readable
cargo nextest list --message-format json

# 7. archive the build artifacts on shard 0, replay on shards 1..N
#    (skip the cargo build on every shard)
cargo nextest archive --archive-file target/nextest-archive.tar.zst
cargo nextest run --archive-file target/nextest-archive.tar.zst \
  --partition hash:2/4
```

Example `.config/nextest.toml` for a CI profile with strict
timeouts and JUnit output:

```toml
[profile.default]
slow-timeout = { period = "60s", terminate-after = 2 }

[profile.ci]
fail-fast = false
retries = 2
slow-timeout = { period = "120s", terminate-after = 3 }
junit = { path = "target/nextest/ci/junit.xml" }
```

## Niche It Fills

**Process-per-test, parallel-by-default, CI-aware test runner
for Rust.** `cargo test` runs all tests in *one* process
per test binary, threaded; one `panic!` in a test that
captures global state can take the rest down, output is
interleaved across threads, there is no built-in retry, no
partition / shard support, no JUnit output, and no
machine-readable test list. `cargo nextest` fixes every one
of those without changing how tests are written — the same
`#[test]` / `#[tokio::test]` / `#[rstest]` attributes, the
same `cargo build` artifacts, just a smarter runner on top.

## Why use it

Three things `cargo nextest` does that `cargo test` does not,
that pay back the switching cost on the first flaky-test
debug:

1. **Process-per-test isolation.** Each test runs in its own
   subprocess of the test binary. A `panic!` that aborts the
   process, a test that calls `std::process::exit`, a test
   that corrupts a `static mut` — none of them can cascade
   into other tests in the same binary. The blast radius of a
   bad test is exactly that test.
2. **First-class CI ergonomics.** `--retries N`, `--partition
   hash:K/N` for sharding without coordination, `--profile
   ci` to flip a whole config block, `junit = { path = ... }`
   to emit XML that GitHub Actions / Buildkite render as a
   native flake / failure report. None of these need wrapper
   shell scripts; they are flags + config.
3. **Machine-readable everything.** `cargo nextest list
   --message-format json` enumerates every test as JSON;
   `cargo nextest run --message-format libtest-json`
   streams structured per-test events. An LLM agent can
   parse "which tests failed and what was the panic message"
   without scraping ANSI-colored console output.

For an agent-driven dev loop ("change file, run tests,
diagnose failure, change file again"), the process-per-test
isolation eliminates a whole class of "the previous run's
poisoned mutex" red-herring failures, and the JSON output
gives the agent a clean failure signal to feed back into the
LLM call.

## Vs Already Cataloged

- **Vs `cargo test` builtin (not cataloged):** baseline.
  `cargo nextest` is a strict superset of features for the
  test-running step (it does not replace `cargo test --doc`
  for doctests — run `cargo test --doc` separately, or use
  `cargo nextest run` plus a follow-up `cargo test --doc` in
  CI).
- **Vs [`bacon`](../bacon/) (cataloged):** orthogonal. `bacon`
  is a *watcher* — it re-runs `cargo` commands on file change
  and renders the result as a TUI. `bacon` can be configured
  to call `cargo nextest run` instead of `cargo test`; the
  two compose. Pick `bacon` for the "save → see test result"
  loop on the laptop, pick `cargo nextest` for the actual
  test execution (in `bacon` *or* in CI).
- **Vs [`hyperfine`](../hyperfine/) (cataloged):** orthogonal.
  `hyperfine` is a benchmark / timing tool ("how fast does
  this command run?"). `cargo nextest` is the test runner.
  Use `hyperfine 'cargo nextest run'` to compare a refactor's
  test wall time, but the two are not substitutes.
- **Vs `cargo test --jobs N` (not cataloged):** that flag
  controls *build* parallelism, not test parallelism. `cargo
  test`'s test parallelism is intra-process threads only;
  `cargo nextest` adds inter-process parallelism on top.

## Caveats

- **Does not run doctests.** `cargo nextest run` runs unit
  and integration tests (the `#[test]` attribute on functions
  inside `src/` and `tests/`); doctests live in `///` comment
  blocks and are run by `cargo test --doc`. Most CI configs
  end up calling both. The
  [docs page on doctests](https://nexte.st/docs/integrations/test-helpers/)
  spells this out.
- **Process-per-test means more startup overhead per test.**
  For suites with thousands of trivial tests, the extra fork
  / exec per test is measurable (typically a few seconds
  added to a 30-second suite). For typical suites with
  meaningful per-test setup the isolation is free; for
  micro-benchmark-shaped suites, profile before assuming a
  speedup.
- **`--no-capture` semantics differ from `cargo test`.**
  `cargo test -- --nocapture` interleaves stdout from all
  parallel tests; `cargo nextest run --no-capture` forces
  serial execution so the live stream is readable. If you
  want parallel + visible logs, write to a per-test file
  instead.
- **Not all `cargo test` flags map 1:1.** `--test-threads`
  is `--test-threads` in nextest too, but a few obscure
  `libtest` flags after `--` are no-ops. The migration guide
  covers the deltas; for typical suites the migration is
  literally `s/cargo test/cargo nextest run/`.
- **JUnit XML output is opt-in via the profile.** Setting
  `junit.path = "..."` in `.config/nextest.toml` under a
  named profile and invoking with `--profile <name>` is the
  contract; CI configs that just set an env var get nothing.
