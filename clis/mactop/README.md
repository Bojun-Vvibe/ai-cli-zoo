# mactop

> **A terminal-based system monitor purpose-built for Apple
> Silicon Macs** — surfaces the CPU / GPU / ANE power draw,
> per-core E-cluster and P-cluster frequency residency, memory
> bandwidth, and integrated-GPU utilization that generic tools
> like `top`, `htop`, or `bottom` cannot read because the
> underlying counters live behind Apple's `powermetrics`
> private API instead of `/proc` or `sysctl` keys. Pinned to
> **v0.2.7** ([LICENSE](https://github.com/context-labs/mactop/blob/main/LICENSE),
> MIT).

Source: <https://github.com/context-labs/mactop>

## TL;DR

`mactop` is the M-series equivalent of `nvtop` for NVIDIA GPUs:
a single-binary TUI that you launch with `sudo mactop` and
that immediately gives you a live per-second breakdown of
where the silicon is actually spending its budget — Performance
cores vs Efficiency cores, integrated GPU residency, Apple
Neural Engine activity, package power in watts, and memory
bandwidth in GB/s. Under the hood it shells out to the
system-bundled `powermetrics(1)` (which is why it needs root)
and parses the plist output into normalized rows, so it
captures counters that no userland API exposes — there is no
`sysctl` or IOKit interface that gives you ANE wattage or
per-cluster frequency residency on macOS, only `powermetrics`,
and `powermetrics` itself prints a 200-line plist every
sample interval that is impossible to read live. `mactop`
collapses that into a sparkline-and-bargraph TUI, refreshes
at 1 Hz by default (configurable), and exits cleanly when you
hit `q`. There is no daemon, no config file, no telemetry —
the binary is ~6 MB, written in Go, and the whole project is
~2 KLOC.

## When to choose

- **You're profiling thermal or power behaviour on an M1 / M2
  / M3 / M4 Mac** — compiling a large project, running local
  inference (Ollama, MLX), encoding video, training a small
  model — and you need to see in real time which cluster
  (E-core, P-core, GPU, ANE) is actually consuming the budget.
  `top` will show you 800% CPU but tell you nothing about
  whether the GPU is idle, whether ANE is being hit, or how
  many watts the package is pulling.
- **You're debugging "why is my Mac hot / why are the fans
  spinning"** — `mactop` shows package wattage and per-cluster
  frequency at a glance; if the GPU is sitting at 1.4 GHz
  drawing 8 W, you know it's not the JS bundler that's the
  problem.
- **You want a screenshot-friendly TUI for a benchmark write-up**
  — bargraphs and sparklines are publication-ready in a way
  that `powermetrics` plist dumps are not.
- **You're comparing local model inference backends** (MLX vs
  llama.cpp Metal vs CPU-only) and need to confirm which
  accelerator each backend is actually hitting — MLX should
  light up the GPU + ANE columns, llama.cpp Metal should hit
  GPU only, CPU-only should leave both at zero.

## When NOT to choose

- **You're on Intel Mac, Linux, or Windows** — `mactop` only
  works on Apple Silicon. Use [`bottom`](../bottom/),
  [`btop`](../btop/), [`zenith`](../zenith/),
  [`nvtop`](https://github.com/Syllo/nvtop), or
  [`procs`](../procs/) instead.
- **You need historical data** — `mactop` is live-only. It
  does not log to disk, does not export Prometheus metrics,
  does not have a JSON output mode. For longitudinal data,
  drive `powermetrics` directly into a file and post-process.
- **You can't grant `sudo`** — `powermetrics` requires root,
  so `mactop` does too. There is no rootless mode and there
  cannot be one without Apple opening the counter API.
- **You want process-level attribution** — `mactop` shows
  *system* counters, not "which PID is causing the GPU
  spike". For per-process breakdown use Activity Monitor or
  `top -stats pid,command,cpu,gputime`.

## Install

```sh
# macOS via Homebrew (recommended)
brew install mactop

# from source (Go 1.21+)
go install github.com/context-labs/mactop@v0.2.7

# from a release binary
curl -LO https://github.com/context-labs/mactop/releases/download/v0.2.7/mactop_Darwin_arm64.tar.gz
tar xzf mactop_Darwin_arm64.tar.gz
sudo install mactop /usr/local/bin/

# verify
sudo mactop --version    # mactop version v0.2.7
```

## What it does

- Live TUI: per-cluster CPU frequency residency (E-cores,
  P-cores), package power in watts, GPU utilization and
  frequency, ANE power draw, memory bandwidth GB/s.
- Sparklines for each metric over the last N samples (default
  ~30 s window).
- Hotkeys: `q` quit, `r` refresh interval cycle, `c` color
  scheme cycle, `p` toggle party mode (rotates colors per
  sample for demo recordings).
- `--interval <sec>` to change sample rate.
- `--color <name>` to pick a color preset for screenshots.
- No config file, no daemon, no network egress — single Go
  binary that wraps `powermetrics`.

## pew-related use cases

- **Local LLM inference profiling**: launch `mactop` in one
  pane and Ollama / llama.cpp / MLX in another, watch
  whether the model actually engages GPU+ANE or sits on the
  P-cores burning battery for no reason.
- **CI runner triage on self-hosted M-series**: when an Xcode
  build mysteriously slows down, `mactop` will tell you
  whether thermal throttling kicked in (P-core frequency
  drops while package power stays flat = thermal cap).
- **Battery-life debugging**: a runaway Electron app that
  pegs an E-core at 100% costs ~0.5 W; a runaway tab on the
  GPU costs ~5 W. `mactop` makes the difference visible
  before you reach for Activity Monitor.
- **Demo recording for a perf write-up**: pair with
  [`asciinema`](../asciinema/) or
  [`freeze`](../freeze/) to capture a clean profile-vs-baseline
  comparison in a blog post.

## Comparable tools

- **[`bottom`](../bottom/)** — cross-platform TUI; on Apple
  Silicon it shows CPU% and memory but cannot read ANE,
  per-cluster residency, or package wattage.
- **[`btop`](../btop/)** — gorgeous TUI; same limitation as
  `bottom` on Apple Silicon (no ANE, no cluster split).
- **[`zenith`](../zenith/)** — adds disk + network historical
  graphs but again no Apple-specific counters.
- **`asitop`** ([upstream](https://github.com/tlkh/asitop)) —
  the original Python prior art that `mactop` is spiritually
  modeled on. `asitop` requires a Python runtime, breaks on
  newer macOS releases when its plist parser falls behind,
  and prints a less polished UI. `mactop` is a single Go
  binary with active releases (v0.2.7 in 2025-12).
- **`macmon`** ([upstream](https://github.com/vladkens/macmon)) —
  newer Rust competitor in the same niche; smaller scope, no
  ANE breakdown. Worth knowing about as the "minimal" option.
- **Apple's own `powermetrics(1)`** — the source-of-truth CLI
  that all of the above wrap. Use it directly when you need
  to log to a file or post-process; use `mactop` when you
  want a human-readable live view.
