# hyperfine

- **Repo:** https://github.com/sharkdp/hyperfine
- **Version:** v1.20.0 (latest stable, 2025-11)
- **License:** MIT / Apache-2.0 dual ([LICENSE-MIT](https://github.com/sharkdp/hyperfine/blob/master/LICENSE-MIT), [LICENSE-APACHE](https://github.com/sharkdp/hyperfine/blob/master/LICENSE-APACHE))
- **Language:** Rust
- **Install:** `brew install hyperfine` · `cargo install --locked hyperfine` · `apt install hyperfine` · binary releases on the GitHub release page

## What it does

`hyperfine` is a statistically rigorous command-line benchmarking tool from
the same author as `bat`, `fd`, and `hexyl`. You hand it one or more shell
commands and it runs each many times, measures wall-clock + user + system
time, runs warmup iterations, detects outliers caused by GC pauses or disk
cache effects, and reports mean ± standard deviation along with min/max and
a relative-speedup ratio between commands. The defaults are conservative
and correct: it auto-picks a run count based on observed variance (10
minimum, more if noise is high), runs `--warmup N` iterations whose timings
are discarded so you measure steady-state rather than cold-start behaviour,
clears the OS file cache between runs with `--prepare 'sync; echo 3 > /proc/sys/vm/drop_caches'`
when you want cold-cache numbers, and warns loudly when results look
suspicious (large outliers, statistical noise > signal). Output formats
include a Markdown table for pasting into a PR, JSON for downstream
tooling, CSV for spreadsheets, and asciidoc for docs. Parameter sweeps
(`--parameter-scan threads 1 16` or `--parameter-list compiler gcc,clang`)
turn it into a one-shot grid runner — feed it a command template with
`{threads}` placeholders and it benchmarks the full cartesian product.
The companion Python script `scripts/plot_whisker.py` shipped in the repo
turns the JSON export into matplotlib whisker plots without writing any
glue code.

## When to pick it / when not to

Pick `hyperfine` whenever you would otherwise type `time foo` three times
and eyeball the numbers. The bar to clear is low: as soon as you care
whether change A is actually faster than change B, the human eyeball is the
worst tool in the chain — `time` reports a single noisy sample, `hyperfine
foo bar` runs each 10+ times with warmup and prints "Command `bar` ran
1.43 ± 0.04 times faster than `foo`" with a real confidence interval. It
is the right tool for comparing two implementations of the same CLI, two
flag combinations of the same binary, two interpreters running the same
script, two compilers building the same code, or for tracking performance
regressions across git commits in CI. The `--export-json` + parameter scan
combo makes it the sane substrate for "build a perf chart for the README"
without rolling your own loop. Pair with [`tokei`](../tokei/) for
codebase-size context, with [`procs`](../procs/) when you need to see what
else is running during a benchmark, and with [`watchexec`](../watchexec/)
to re-benchmark on every save during a perf-tuning session.

Skip it for **microbenchmarks under ~1 ms** — process-startup cost
dominates and you want an in-process harness like `criterion` (Rust),
`google-benchmark` (C++), `pytest-benchmark` (Python), or `hotspot` /
`perf` for instruction-level work. Skip it when you need flame graphs,
call-stack attribution, or "where is the time going inside the process" —
that is a profiler's job (`samply`, `perf`, `Instruments.app`,
`py-spy`), not a wall-clock harness. Skip it for benchmarks that need to
share warmed-up state across runs (a long-lived database connection, a
loaded ML model) — `hyperfine` re-execs the command each iteration by
design, which is the whole point for CLI tools but the wrong shape for
"measure 10 000 RPC calls inside one process". And skip it on shared CI
runners with noisy neighbours unless you crank `--warmup` and `--runs`
high and accept that the confidence intervals will be wide.

## Example invocations

```bash
# Compare two commands; auto-picks run count, prints relative speedup
hyperfine 'fd pattern' 'find . -name "*pattern*"'

# Warmup runs, useful when the first run pays a JIT or cache cost
hyperfine --warmup 3 'cargo build --release'

# Cold-cache benchmark (Linux): drop the page cache before each run
hyperfine --prepare 'sync; sudo sh -c "echo 3 > /proc/sys/vm/drop_caches"' \
  'rg pattern /var/log' 'grep -r pattern /var/log'

# Parameter scan: time `make -j N` for N in 1..16
hyperfine --parameter-scan threads 1 16 'make -j {threads}'

# Parameter list: same script under two interpreters
hyperfine --parameter-list py python3,pypy3 '{py} fib.py 35'

# Export Markdown table for a PR description
hyperfine --export-markdown bench.md \
  './bin/v1 input.json' './bin/v2 input.json'

# JSON export for downstream charting
hyperfine --export-json results.json \
  --warmup 3 --runs 50 \
  './target/release/myapp --threads 1' \
  './target/release/myapp --threads 8'

# Track a regression in CI: fail if v2 is more than 5% slower than v1
hyperfine --export-json out.json './v1' './v2' \
  && jq -e '.results[1].mean / .results[0].mean < 1.05' out.json
```
