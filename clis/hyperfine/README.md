# hyperfine

> **Command-line benchmarking tool** — a single Rust binary that
> runs any shell command N times, controls for warmup runs and
> disk cache, detects statistical outliers, and prints a
> mean ± σ summary with a min / max range and a relative-speed
> ratio when you pass multiple commands. Exports raw timings as
> JSON / CSV / Markdown / AsciiDoc so the result lands directly
> in a PR description, a regression dashboard, or a
> `criterion`-style follow-up plot. Pinned to **v1.20.0** (commit
> `327d5f4d9107141929f67f062bf9ef59f98b7399`,
> [LICENSE-MIT](https://github.com/sharkdp/hyperfine/blob/master/LICENSE-MIT),
> MIT — also dual-licensed Apache-2.0 via `LICENSE-APACHE`).

Source: <https://github.com/sharkdp/hyperfine>

## TL;DR

`hyperfine` is what you reach for when "is this faster?" needs an
answer that is not "I ran it once and it felt snappier." Pass one
command and it runs at least 10 times (auto-tuned by initial-run
duration), prints `Time (mean ± σ): 142.3 ms ± 4.1 ms` with a
min / max envelope and warns you about outliers. Pass two or more
commands and it benchmarks each, then prints a sorted ranking with
relative-speed ratios (`Command A ran 1.42 ± 0.06 times faster
than Command B`). `--warmup N` runs N untimed iterations first to
populate disk / page cache so the first cold run does not skew the
mean. `--prepare 'sync; echo 3 > /proc/sys/vm/drop_caches'` runs a
setup command between every measured run for the *opposite*
posture (cold cache benchmarking). `--parameter-scan threads 1 16
-D 1` sweeps a parameter across a range and emits one row per
value, ready for `--export-json` into a plot. Default output is a
human-readable terminal table; `--export-markdown bench.md` drops
a copy-pasteable table into your PR.

## Install

```bash
# Homebrew (macOS / Linux)
brew install hyperfine

# Cargo
cargo install --locked hyperfine

# Linux package managers
# Arch: pacman -S hyperfine
# Nix: nix-env -iA nixpkgs.hyperfine
# Fedora: dnf install hyperfine
# Debian / Ubuntu: download .deb from releases page

# Windows
# scoop install hyperfine
# choco install hyperfine
# winget install hyperfine

# from a release tarball (any OS)
curl -Lo hyperfine.tar.gz "https://github.com/sharkdp/hyperfine/releases/download/v1.20.0/hyperfine-v1.20.0-aarch64-apple-darwin.tar.gz"
tar xf hyperfine.tar.gz --strip-components=1 hyperfine-v1.20.0-aarch64-apple-darwin/hyperfine
sudo install hyperfine /usr/local/bin/

# verify
hyperfine --version    # hyperfine 1.20.0
```

Zero config. The first invocation prints a summary; that is the
whole UX.

## License

MIT (or Apache-2.0 at your option) — see
[LICENSE-MIT](https://github.com/sharkdp/hyperfine/blob/master/LICENSE-MIT)
and [LICENSE-APACHE](https://github.com/sharkdp/hyperfine/blob/master/LICENSE-APACHE).
Permissive dual license; redistribute freely, no attribution
required for binaries.

## One Concrete Example

```bash
# 1. compare two implementations of the same task
hyperfine --warmup 3 \
  'rg --no-mmap -c "TODO" .' \
  'grep -rc "TODO" .'
# Benchmark 1: rg --no-mmap -c "TODO" .
#   Time (mean ± σ):      31.2 ms ±   1.4 ms    [User: 18.1 ms, System: 12.7 ms]
#   Range (min … max):    29.4 ms …  35.8 ms    91 runs
# Benchmark 2: grep -rc "TODO" .
#   Time (mean ± σ):     412.0 ms ±   8.6 ms    [User: 388.2 ms, System: 22.1 ms]
#   Range (min … max):   401.1 ms … 428.3 ms    10 runs
# Summary
#   'rg --no-mmap -c "TODO" .' ran
#    13.21 ± 0.66 times faster than 'grep -rc "TODO" .'

# 2. parameter scan: how does runtime scale with thread count?
hyperfine --parameter-scan threads 1 8 -D 1 \
  'cargo build --release -j {threads}' \
  --export-json scan.json --export-markdown scan.md

# 3. cold-cache benchmark on Linux (drop page cache before each run)
sudo hyperfine --prepare 'sync; echo 3 > /proc/sys/vm/drop_caches' \
  'cat large.bin > /dev/null'

# 4. CI-shaped invocation: fail if regression exceeds threshold
hyperfine --warmup 5 --runs 30 \
  --export-json bench.json \
  './build/main --quick'
jq '.results[0].mean' bench.json    # 0.142 -> compare against baseline in CI
```

## Niche It Fills

**Statistically rigorous wall-clock benchmarking with one
command, no harness code.** Writing a `criterion` benchmark in
Rust or a `pytest-benchmark` harness in Python takes 30 minutes
and only measures things you can call from inside that language;
`hyperfine` benchmarks any shell command (a build, a
`docker run`, a `curl`, a `cargo test`, an LLM CLI invocation
with `--no-cache`) with warmup, statistical-outlier detection,
and a comparison ranking, in five seconds. The hard problem is
*not* timing one run — it is "did this change make things faster
or did I just get lucky on the dice roll" — and `hyperfine` is
the cheapest tool that gives you the σ to answer that question.

## Why use it

Three things `hyperfine` does that `time` / `bash time loop` do
not, that pay back the switching cost:

1. **Auto-tuned run count + warmup + outlier detection.** `time`
   runs once and prints user / sys / wall; you have to write the
   loop, compute the mean and σ, decide how many warmup runs to
   discard, and notice when an outlier (a GC pause, a context
   switch, a disk-cache cold miss) skews the mean. `hyperfine`
   defaults to "at least 10 runs, at least 3 seconds total," runs
   `--warmup N` discarded warmup iterations first, and prints a
   warning when it detects outliers (`Warning: Statistical
   outliers detected. Consider re-running with --warmup`).
2. **Parameter scans + multi-command comparison as flags.**
   `--parameter-scan threads 1 16 -D 1` and `--parameter-list
   compiler clang,gcc,zig-cc` are first-class — you do not write
   a shell loop and a CSV writer, you get one row per parameter
   value and `--export-json` / `--export-csv` ready to plot. With
   multiple positional commands, the ranking + relative-speed
   ratios come for free; with one command and `--parameter-scan`,
   the parameter sweep is the comparison.
3. **Export shapes that fit a PR or a CI gate.**
   `--export-markdown bench.md` drops a copy-pasteable table into
   the PR description; `--export-json bench.json` is `jq`-able for
   regression gates (compare mean against a baseline file, fail
   the build if the ratio exceeds a threshold). `criterion` /
   `pytest-benchmark` give you better statistics on a single
   language; `hyperfine` gives you portable, language-agnostic
   numbers in a shape your reviewers can paste.

For an LLM-CLI workflow where you want to know "is this prompt
template + model combo actually faster end-to-end" or "did
swapping `requests` for `httpx` change cold-start latency by more
than noise," `hyperfine --warmup 3 --runs 30 'cli-a "..."' 'cli-b
"..."'` answers it in 30 seconds with a defensible σ.

## Vs Already Cataloged

- **Vs [`pew-insights`](https://github.com/anomalyco/pew-insights)
  (not cataloged):** orthogonal layer. `pew-insights` analyses
  *recorded* token / latency telemetry from existing LLM CLI
  runs (Hurst, DFA, sample entropy on observed time-series);
  `hyperfine` *generates* the time-series by running a command
  N times under controlled warmup. Use `hyperfine` to produce
  the data, then feed `--export-json` rows into a downstream
  analysis.
- **Vs [`ttok`](../ttok/):** orthogonal — `ttok` is a token
  counter (pre-flight: "will this fit in the context window")
  and `hyperfine` is a runtime benchmarker (post-hoc: "how long
  did it take, with σ"). They sit at opposite ends of the same
  pipeline.
- **Vs [`bottom`](../bottom/):** orthogonal — `bottom` shows
  *live* CPU / memory / network as a continuous TUI (you watch
  a long-running process), `hyperfine` runs a finite command N
  times and prints summary statistics (you measure a discrete
  workload). Pair them when benchmarking a long-running server:
  `bottom` for the live view, `hyperfine` for the request
  latency distribution.
- **Vs `time` / `gtime` / `bash time loop` (not cataloged):**
  `time` is one run with no σ; the bash loop wrapper around
  `time` is what `hyperfine` replaces. Pick `time` for "I just
  want a single wall-clock number"; pick `hyperfine` for "I want
  a confidence interval and a comparison."

## Caveats

- **Wall-clock only by default.** `hyperfine` measures user /
  system / wall on Linux + macOS but does not break down into
  per-syscall or per-function profiles — it answers "how long"
  not "where in the code." Pair with `perf` / `samply` /
  `cargo-flamegraph` when the answer is "slow, why?"
- **Sub-millisecond commands need `-N`.** By default `hyperfine`
  spawns a shell to interpret the command (so `&&`, `|`, `;` work),
  which adds ~1-2 ms of shell startup to every run. For commands
  that finish in under 5 ms, pass `-N` (no intermediate shell)
  to remove that floor; the trade-off is you lose pipe / redirect
  / `&&` parsing in the command string.
- **`--warmup` and `--prepare` are different things.** Warmup
  runs are timed-but-discarded iterations that populate caches;
  `--prepare` is a setup command run *between* measured iterations
  (drop caches, restart a service, restore a fixture). Mixing them
  up gives you the wrong cache posture.
- **Parameter scans multiply runs by parameter count.** A scan
  over 1..16 with default 10 runs each is 160 invocations — for a
  3-minute build, that is an 8-hour benchmark. Use `--runs 5` and
  `--time-unit second` for slow commands.
- **Statistical outlier warning is heuristic.** `hyperfine` flags
  outliers when σ exceeds a threshold relative to the mean; this
  catches obvious GC pauses but not subtle bimodal distributions
  (e.g. a JIT warmup that flips between two stable means). Export
  to JSON and inspect the histogram for high-stakes decisions.
- **Windows process-creation overhead is higher than Linux /
  macOS.** Sub-10 ms commands on Windows have a wider σ baseline;
  prefer `--runs 50` or wrap multiple iterations into the command
  itself (e.g. a tight loop in the program under test) on
  Windows.
