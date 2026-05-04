# gpustat

> **One-screen GPU summary CLI** — a small Python wrapper
> around NVML (`pynvml`) that prints one line per NVIDIA GPU
> (index, name, temperature, fan %, utilisation %, used /
> total memory, power-draw / power-cap, plus a colourised
> bar of the running processes' PIDs + per-process MiB) in
> the shape of `nvidia-smi` minus the 30-line repeating
> header — pinned to **v1.1.1** (commit
> [`265c50f6`](https://github.com/wookayin/gpustat/commit/265c50f65930c591f70103b79794124f46697377),
> [LICENSE](https://github.com/wookayin/gpustat/blob/v1.1.1/LICENSE),
> MIT).

Source: <https://github.com/wookayin/gpustat>

## TL;DR

`nvidia-smi` is the canonical NVIDIA GPU monitor and ships
with the driver — but it prints a 30-line table dominated by
a fixed header (driver version, CUDA version, table border,
process-table border) where the actual answer to "is this GPU
busy and who is on it" is two lines. `gpustat` is the
one-line-per-GPU shape: `[0] NVIDIA RTX 4090 | 67'C, 78 %,
3.2 / 24.0 GiB | bojun:python/12345(3.1G)` is *one* line per
device on a host with N GPUs, with ANSI colour for
temperature / utilisation / memory-pressure thresholds so a
glance picks out the hot card or the OOM-near card without
parsing numbers.

The killer property is **NVML-direct**: `gpustat` links
`pynvml` against the running driver's `libnvidia-ml.so` and
asks the kernel module the same questions the `nvidia-smi`
binary asks — so the values are the authoritative ones from
the driver, not parsed from `nvidia-smi`'s text output (the
common shape of older `nvidia-smi`-wrapper scripts that break
when a driver release rearranges the table). Process
attribution joins NVML's per-PID memory query against `/proc`
to surface the *user* + *command* per process, so "whose
training run is hogging GPU 3" answers in one line.

`gpustat` is NVIDIA-only (NVML is NVIDIA's library); for AMD
ROCm hosts use `rocm-smi` directly, for Apple Silicon use
`asitop` / `mactop`, for the cross-vendor "tell me about the
GPU hardware" question see [`cpufetch`](../cpufetch/)'s
sibling project `gpufetch`.

## Install

```bash
# pipx (recommended — isolated venv, NVML loaded from system driver)
pipx install gpustat

# pip
pip install gpustat

# Homebrew (brings its own pynvml)
brew install gpustat

# Arch Linux
sudo pacman -S gpustat

# Debian / Ubuntu (slightly older)
sudo apt install gpustat

# verify
gpustat --version    # 1.1.1
```

Requires Python 3.6+. The host needs an NVIDIA driver with
NVML available (`nvidia-smi` working is sufficient; gpustat
loads `libnvidia-ml.so` via `pynvml`). Works on any Linux
distro with NVIDIA drivers, on WSL2 with the Windows-side
NVIDIA driver exposing `/usr/lib/wsl/lib/libnvidia-ml.so.1`,
and on Windows Server with the matching NVML DLL — does *not*
work on macOS (Apple dropped NVIDIA driver support after
High Sierra) or on AMD / Intel GPUs.

## Example usage

```bash
# one-shot snapshot
gpustat

# per-GPU process table (the killer flag — adds users + cmds)
gpustat -cup

# watch mode (re-renders every 1 s in place, like top)
gpustat --watch

# every 0.5 s, with full process info, force-colour for tmux
gpustat -cupP --watch -i 0.5 --color

# JSON output for monitoring / scripting
gpustat --json | jq '.gpus[] | {idx:.index, util:."utilization.gpu", mem:."memory.used"}'

# only show GPUs with running processes (skip idle)
gpustat --no-color | awk '$0 ~ /\| .* \(/'
```

Common flags:

- `-c` show command name per process
- `-u` show user per process
- `-p` show PID per process
- `-P` show power-draw / power-cap
- `-f` show fan speed
- `-i N` refresh interval seconds (with `--watch`)
- `--watch` / `-w` continuous render in place (Ctrl+C exits)
- `--json` machine-readable output
- `--debug` print pynvml errors instead of swallowing them
- `--no-color` disable ANSI for log capture

## Why it matters

- **One line per GPU** beats `nvidia-smi`'s 30-line table when
  the host has 1, 2, 4, or 8 GPUs and the question is "which
  ones are busy right now." On an 8-GPU DGX node the gpustat
  output is 8 lines plus a header; the `nvidia-smi` output
  is ~80 lines.
- **Process attribution with user + command** turns "GPU 3 is
  at 95 %" into "GPU 3 is `bojun` running `python train.py`,
  PID 4711, 18.3 GiB" — the right answer for the "who is
  hogging the shared box" triage question without grepping
  `ps` against the per-PID list `nvidia-smi` prints.
- **JSON output** is the cheap monitoring-pipeline integration:
  `gpustat --json` every 30 s into a `jq` filter into
  Prometheus pushgateway / VictoriaMetrics / `vector` is a
  10-line cron without standing up DCGM-Exporter.
- **NVML-direct** values match what the driver believes,
  including per-process memory accounting that `top` /
  `htop` cannot see (GPU memory is not in `/proc/$pid/status`
  the way RAM is).
- **`--watch` mode** re-renders in place at sub-second rates
  on a single short table, the right "leave it open in a
  small tmux pane next to the training-loss plot" footprint.
  Compare to `nvtop` (full-screen TUI with per-GPU
  utilisation graphs — better for *watching* the GPU over
  minutes; gpustat better for *answering* the "what is
  running right now" question in one glance).

## Vs Already Cataloged

- **Vs [`bottom`](../bottom/) / [`btop`](../btop/) /
  [`htop`](../htop/):** orthogonal — those are CPU + RAM +
  IO + (sometimes) GPU dashboards but the GPU panel is
  shallow (one utilisation number, no per-process attribution).
  gpustat is GPU-deep, single-purpose, and shells out cleanly
  into a tmux pane next to a `bottom` pane.
- **Vs [`cpufetch`](../cpufetch/):** orthogonal axes —
  cpufetch is one-shot CPU *identification* (microarchitecture,
  cache, AVX flags); gpustat is recurring GPU *utilisation*
  (busy-or-not, who is on it). Both belong in a
  benchmarking / triage dotfile.
- **Vs `nvidia-smi`:** gpustat is the same data in 1/N the
  vertical space, with user+command joined in by default,
  with ANSI colour, and with `--json` as a first-class
  output. `nvidia-smi` is still the right pick for the
  rarely-needed bottom-half data (ECC error counters, PCIe
  link width, MIG slice config, `--query-gpu` extreme
  flexibility) — the two compose: gpustat for the daily
  glance, `nvidia-smi -q` when you need everything.
- **Vs `nvtop` / `nvitop`:** orthogonal — those are
  full-screen TUIs with per-GPU utilisation history graphs
  (the right tool for *watching* a training run); gpustat
  is the one-line snapshot (the right tool for *answering*
  "who is on this card" or for piping into a monitoring
  pipeline). Many operators install both.

## License

MIT — see
[LICENSE](https://github.com/wookayin/gpustat/blob/v1.1.1/LICENSE).
Permissive; no obligations on downstream users beyond
attribution.

## Caveats

- **NVIDIA-only.** AMD users want `rocm-smi`, Apple Silicon
  users want `asitop` / `mactop`, Intel discrete GPU users
  want `intel_gpu_top`. There is no cross-vendor unified
  GPU monitor in this catalog yet; gpustat fills the
  NVIDIA slot specifically.
- **Requires NVML at runtime.** If `nvidia-smi` does not
  work on the host (driver missing, container without
  `--gpus all`, WSL2 without the right `/usr/lib/wsl/lib`
  bind), gpustat will not work either. Run `nvidia-smi`
  first to verify.
- **Last upstream release was 2023-08-22 (v1.1.1).** The
  project is in maintenance — `master` has small commits
  but no new tag in over two years. The NVML API surface
  it depends on is stable across recent driver versions
  (R535, R545, R550, R555, R560, R565), so this is "stable
  not stalled" rather than abandoned.
- **Per-process memory accounting can be empty in
  containers.** When gpustat runs inside a container that
  does not have access to host `/proc`, the per-PID column
  shows the PID but cannot resolve the user / command.
  Run gpustat on the *host* (or pass `--pid-host` to the
  container) for full attribution.
- **MIG (Multi-Instance GPU) on A100/H100 surfaces each
  MIG slice as a separate GPU index** — correct behaviour,
  but the topology may surprise an operator expecting one
  line per physical card.

## As of

2026-05-04. Upstream tag `v1.1.1` (2023-08-22). NVML-backed
queries are stable across NVIDIA driver branches R535+;
re-verify if a future driver removes a deprecated NVML
field that gpustat queries.
