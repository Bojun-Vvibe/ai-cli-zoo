# bencher

> **Continuous benchmarking suite that catches performance
> regressions in CI the same way unit tests catch correctness
> regressions.**
> A Rust workspace shipping the `bencher` CLI plus a self-hostable
> server: the CLI runs your existing benchmark harness
> (`cargo bench`, `pytest-benchmark`, Google Benchmark, JMH, k6,
> custom JSON), parses the results, and POSTs them to a project on
> the server, which stores a per-branch time series, runs
> statistical regression checks (t-test / IQR / percentile
> thresholds), and fails the PR if a metric drifts beyond its
> baseline. Pinned to **v0.6.4** (released 2026-05-02, dual SPDX:
> `Apache-2.0 OR MIT`, with a `Bencher Plus` carve-out for paths
> named `plus/` —
> [LICENSE.md](https://github.com/bencherdev/bencher/blob/main/LICENSE.md),
> [license/LICENSE-MIT](https://github.com/bencherdev/bencher/blob/main/license/LICENSE-MIT),
> [license/LICENSE-APACHE](https://github.com/bencherdev/bencher/blob/main/license/LICENSE-APACHE)).

Source: <https://github.com/bencherdev/bencher>

## TL;DR

`bencher run --project myapp --branch main "cargo bench"` wraps your
benchmark command, captures its output, parses it into Bencher's
metric format, and uploads to the server with the current commit
SHA + branch. On a PR, the same command runs against the PR branch
and the server compares the result to the target branch's recent
history; if any metric exceeds the configured threshold (e.g.
"latency p95 must not regress more than 2 sigma vs. the trailing
30 commits on main"), `bencher run` exits non-zero and the PR check
goes red.

## Install

```sh
# macOS / Linuxbrew
brew install bencherdev/bencher/bencher

# Or one-liner installer
curl -L https://bencher.dev/download/install-cli.sh | sh

# Or from source
cargo install --locked bencher_cli@0.6.4

# Verify
bencher --version    # 0.6.4
```

The server can be self-hosted (`bencher up` via Docker Compose) or
used as the hosted SaaS at <https://bencher.dev>.

## License

Dual `Apache-2.0 OR MIT` for the open core (CLI + server + UI),
plus a separate `Bencher Plus License` for any code under a
directory named `plus/` (paid-tier features). Paths matter: as
long as you build *without* the `plus` Cargo feature and avoid
`plus/`-rooted code, the binary you ship is pure Apache/MIT and
unrestricted. The CLI used in CI is fully under the open licence.

## Primary use case

You have a hot-path service or library where a 5% throughput drop
or a 20 ms p99 increase is a real bug, but your existing CI only
fails on incorrect results, not on slow ones. You already write
`#[bench]` / `criterion` benchmarks, but no one looks at the
numbers because there is no baseline. Add `bencher run` to CI:
the server stores the time series, draws the trend line, and
fails the PR when a regression actually happens. Reviewers stop
re-running `cargo bench` on their laptop to "eyeball it."

## What it competes with

- **Hand-rolled "compare two `cargo bench` runs" scripts** — what
  most teams have. Works for one repo and one engineer who
  remembers to run it; rots within a quarter. `bencher` is the
  productized version.
- **[`hyperfine`](../hyperfine/) in CI** — great for ad-hoc
  command-line benchmarking, no time-series store, no PR check.
  Pair them: `hyperfine --export-json` → `bencher run --adapter
  json …` is a supported flow.
- **Codspeed / Datadog Continuous Profiler / NYRKIÖ** — hosted
  commercial competitors. `bencher` stands out by being
  open-source and self-hostable; pick a hosted commercial product
  if you want zero ops and don't mind shipping perf data
  off-prem, pick `bencher` when the perf data itself is
  sensitive or you want to own the server.
- **[`criterion.rs`](https://github.com/bheisler/criterion.rs) /
  `pytest-benchmark` / `JMH` alone** — they produce numbers, they
  don't gate PRs. `bencher` is the layer that turns local
  benchmark output into a CI gate with history.
- **[`grafana-k6`](../k6/) / [`vegeta`](../vegeta/) /
  [`bombardier`](../bombardier/)** — load generators, useful as
  *inputs* to `bencher run` (they have JSON adapters), not
  replacements.

## AI-native angle

Coding agents that "make this faster" or "refactor without
regressing" need a numerical truth oracle. Without one, they
will happily ship a "cleaner" rewrite that doubles allocations
or quietly serializes a hot loop:

- **Perf is a CI signal, not a vibe.** With `bencher run` on the
  PR check, an agent that finishes a refactor sees the same red
  X for "you regressed throughput by 8%" that it sees for "you
  broke a test." No need to teach the agent to run benchmarks
  manually — the gate enforces it.
- **Threshold reasoning is structured.** `bencher` thresholds
  (t-test, percentile, static) are configured per metric per
  branch; an agent can read the threshold config, reason about
  "is this regression within tolerance," and either accept it
  with justification in the PR body or back out the change.
- **Time-series API.** The server exposes per-metric history as
  JSON. An agent debugging "why did p95 jump on main last
  Tuesday" can query the API for the offending commit range and
  bisect against the time series instead of re-running benches
  blindly.
- **Adapter list is broad.** Rust (`cargo bench`, `criterion`,
  `iai`), C/C++ (Google Benchmark), Python
  (`pytest-benchmark`), Go (`go test -bench`), Java (JMH),
  shell (`hyperfine`), or raw JSON. An agent operating across a
  polyglot monorepo can use the same gate everywhere.
- **Bench-as-spec.** Telling an agent "speed up `parse_event`
  by 20% on the `criterion::parse_event` bench, do not regress
  any other bench by more than 2%" is a precise, machine-checked
  contract. `bencher` is what makes that contract enforceable.

## Caveats

- **CI noise is real.** Shared CI runners have variable CPU
  thermal / neighbour-noise behaviour; tight thresholds will
  flap. Use statistical thresholds (t-test against trailing
  history) rather than absolute deltas, and prefer self-hosted
  runners pinned to consistent hardware for the perf job.
- **Benchmarks have to be deterministic-ish.** A benchmark that
  hits the network, depends on wall-clock, or uses RNG without a
  seed will fail the gate at random. The `bencher` adapter is
  only as good as the harness feeding it.
- **Server is part of the deploy.** Continuous benchmarking
  requires a place to store history. The hosted SaaS works out
  of the box; self-hosting is one Docker Compose stack but it
  *is* a stack you now operate (postgres, server, UI).
- **`plus/` paths are not Apache/MIT.** If you fork the repo and
  redistribute, exclude `plus/` directories or build with the
  `plus` feature disabled to stay inside the open licence. The
  upstream CI builds both modes; read
  [LICENSE.md](https://github.com/bencherdev/bencher/blob/main/LICENSE.md)
  before vendoring.
- **Not a profiler.** `bencher` tells you a metric regressed; it
  does not tell you *why*. Pair with `cargo flamegraph`,
  `samply`, `perf`, `pprof`, or `tracy` for root cause.

## Concrete example

GitHub Actions step that gates a Rust PR on Criterion benches:

```yaml
- name: Run Criterion benches and report to Bencher
  uses: actions/checkout@v4

- run: cargo bench --bench parse -- --output-format bencher
  # produces a bencher-format file the CLI can ingest

- name: bencher run
  env:
    BENCHER_API_TOKEN: ${{ secrets.BENCHER_API_TOKEN }}
  run: |
    bencher run \
      --project acme-parser \
      --branch "${{ github.head_ref || github.ref_name }}" \
      --branch-start-point "${{ github.base_ref }}" \
      --testbed gha-ubuntu-22.04 \
      --adapter rust_criterion \
      --threshold-measure latency \
      --threshold-test t_test \
      --threshold-max-sample-size 64 \
      --threshold-upper-boundary 0.99 \
      --thresholds-reset \
      --err \
      --github-actions "${{ secrets.GITHUB_TOKEN }}" \
      "cargo bench --bench parse -- --output-format bencher"
```

That single step replaces a wiki page that says "remember to
re-run the benchmarks before merging," and turns it into a check
the PR cannot bypass without explicit override.
