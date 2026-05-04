# samply

- **Repo:** https://github.com/mstange/samply
- **Version:** samply-v0.13.1 (latest release, 2025)
- **License:** MIT / Apache-2.0 dual ([LICENSE-MIT](https://github.com/mstange/samply/blob/main/LICENSE-MIT), [LICENSE-APACHE](https://github.com/mstange/samply/blob/main/LICENSE-APACHE))
- **Language:** Rust
- **Install:** `cargo install --locked samply` · `brew install samply` · prebuilt
  binaries on the GitHub release page · `cargo binstall samply`

## What it does

`samply` is a cross-platform sampling profiler that records a stack-walked
profile of any command and uploads (or hosts) the result in the **Firefox
Profiler** web UI. The model is the same as `perf record` on Linux or
`Instruments.app` on macOS — install no agent, instrument no source, just
run `samply record ./your-binary --your-flags` and on Ctrl-C it opens
`https://profiler.firefox.com/` in your browser with the captured stacks
already loaded. The viewer is the killer feature: a flame graph, a
call tree, a marker timeline, source-line attribution when symbols are
present, and a stack chart that scrubs through time — all of it
shareable as a permalink (`samply load profile.json` re-opens it
locally without uploading anywhere). Backed by the same Mozilla team
that ships the Firefox Profiler, so the import path is first-class
rather than a third-party converter.

Three subcommands cover the workflow:

- `samply record <cmd>` — spawn a process and sample it (default 1000 Hz).
- `samply record -p <pid>` — attach to an already-running process by PID.
- `samply load profile.json.gz` — re-open a previously saved profile in the
  viewer without re-running anything.

`--save-only` writes the gzipped JSON profile to disk without launching a
browser, which is the shape you want in CI when you just want to attach a
profile artifact to a PR. `--rate N` overrides the sampling frequency.
`--unstable-presymbolicate` resolves symbols ahead of time so air-gapped
viewers do not need network access. `samply` reads DWARF directly, so a
debug-info-stripped Rust / C / C++ / Go binary needs `RUSTFLAGS=-C
debuginfo=2` (or `-g`) to give you function-level resolution instead of
hex addresses.

## When to pick it / when not to

Pick `samply` when you have a CPU-bound binary and want to know "where is
the time actually going" without the ceremony of `perf` + FlameGraph.pl
or the macOS lock-in of Instruments. The bar to clear is low: as soon as
[`hyperfine`](../hyperfine/) tells you "command B is 1.4× slower than A"
and you cannot guess why, `samply record ./b` and `samply record ./a` are
the next two commands. It is the right tool for profiling Rust / C /
C++ / Go binaries during local perf work, for capturing a one-off
production CPU profile on a Linux box (`samply record -p $(pgrep myapp)`
for 30 seconds), and for attaching a shareable profile permalink to a
GitHub PR — reviewers click the link and get the same flame graph the
author saw, no tooling install required. The macOS support is
production-grade (it is a Mozilla project run by Mozilla profiler
engineers), which is rare among Rust profilers.

Skip it for **wall-clock benchmarks of CLI tools** — that is
[`hyperfine`](../hyperfine/)'s job, and `samply` adds sampling overhead
that distorts those numbers. Skip it for **scripting-language
profiling** — `samply` profiles native code, so a Python script will show
you `python3` and the interpreter's C frames, not your Python functions
(use [`py-spy`](../py-spy/) for that, or `--profile` for Node, or
`pprof` for Go's runtime-aware profiler when you need allocation /
goroutine profiles too). Skip it for **memory profiling, allocation
tracking, or off-CPU analysis** — `samply` is a *CPU* sampling profiler;
`heaptrack`, `bytehound`, `valgrind --tool=massif`, or `dhat` are the
right shape for memory, and `bpftrace` / `perf sched` for off-CPU
waits. And skip it on Windows for sub-millisecond functions — the
sampling rate caps below Linux/macOS due to OS limits.

## AI-native angle

For agents driving long-horizon perf-tuning loops, `samply` slots into
the same observable-result substrate as [`hyperfine`](../hyperfine/) and
[`tokei`](../tokei/):

- **Profile JSON is grep-able.** `samply record --save-only -o
  profile.json.gz ./bin` produces a deterministic file an agent can
  diff between commits to spot which function's self-time grew. Pair
  with `hyperfine --export-json` for the wall-clock numbers and
  `samply` for the attribution.
- **Permalinks are PR artifacts.** A reviewer agent reading the PR
  description sees `before: <profiler-url>` / `after: <profiler-url>`
  and can fetch both, narrate the flame-graph delta, and quote the
  specific function that regressed — no local install required.
- **Attach-by-PID works on running services.** When the agent has SSH
  access to a misbehaving box, `samply record -p $(pgrep myservice)
  --duration 30` captures a profile without restarting the process.

## Alternatives

- **[`perf`](https://www.kernel.org/doc/html/latest/admin-guide/perf.html)
  + [FlameGraph.pl](https://github.com/brendangregg/FlameGraph)** — the
  classic Linux story, full hardware-counter support (cache misses,
  branch mispredicts, IPC), much deeper than `samply` when you need to
  ask "is this CPU-bound or memory-bound?". Cost: Linux-only, two-tool
  pipeline, the FlameGraph SVG is read-only (no scrubbing, no source
  attribution). Pick `perf` for hardware-counter work; pick `samply`
  for everyday CPU profiling with a real interactive viewer.
- **[Instruments.app](https://developer.apple.com/documentation/xcode/profiling-your-app-s-performance)**
  — Apple's first-party profiler, deeply integrated with Xcode and
  the macOS scheduler. Pick `Instruments` for macOS GUI app work where
  you also want Allocation / Leaks / Time Profiler in one place; pick
  `samply` when you want the same data in a browser-shareable format
  and on Linux + Windows too.
- **[`py-spy`](../py-spy/)** — sampling profiler for *Python*, with
  the same "no source changes, attach to a PID" UX. Different layer:
  `py-spy` understands the CPython interpreter and shows you Python
  function names; `samply` shows you C function names. They do not
  overlap; both can run at the same time on the same process.
- **[`flamegraph-rs`](https://github.com/flamegraph-rs/flamegraph)** —
  cargo subcommand that wraps `perf` (Linux) or `dtrace` (macOS) and
  emits an SVG flame graph. Lighter dependency than `samply`'s viewer,
  but the SVG is non-interactive — pick it for one-shot inclusion in
  a static doc, pick `samply` for the live shareable viewer.
- **[`pprof`](https://github.com/google/pprof)** — Go's
  runtime-integrated profiler with first-class allocation / goroutine /
  block / mutex profiles. Different shape: `pprof` requires the
  process to expose a debug endpoint or accept a runtime API call;
  `samply` attaches externally. Pick `pprof` when you are profiling Go
  and want allocation profiles; pick `samply` when you want CPU
  profiling on a non-Go process from outside.
- **[`bpftrace`](https://github.com/bpftrace/bpftrace) /
  [`bcc`](https://github.com/iovisor/bcc)** — eBPF tracing toolkits.
  Different layer: kernel-level dynamic tracing for "why is this
  syscall slow", not user-space CPU sampling. Pair, do not replace.
