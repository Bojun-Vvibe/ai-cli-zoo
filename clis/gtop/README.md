# gtop

> **A terminal system-monitoring dashboard in Node** — `top` /
> `htop` reframed as a multi-pane TUI with sparkline charts for
> CPU, memory, swap, network, and disk on one screen. Pinned to
> **v1.1.5** (homebrew `gtop`),
> [LICENSE](https://github.com/aksakalli/gtop/blob/master/LICENSE),
> MIT.

Source: <https://github.com/aksakalli/gtop>

## TL;DR

`gtop` is what you launch when you want one pane of glass over a
remote box without opening five `tmux` panes. The default layout
is six widgets on one screen: a per-core CPU sparkline grid at
the top, a memory usage line chart and a swap chart on the next
row, a stacked network-throughput chart (rx/tx), a horizontal
filesystem-usage bar list, and a sortable process table at the
bottom. Sort the process table interactively with `p` (PID), `c`
(CPU%), `m` (memory%); type `/` to filter by command name; arrow
keys navigate; `q` exits. Charts redraw via the `blessed` /
`blessed-contrib` ncurses-style stack, so it works fine over plain
SSH and inside `tmux`. It is a Node app (single `npm i -g`
install), not a single static binary like
[bottom](../bottom/) / [zenith](../zenith/), but the trade is
that it runs anywhere Node runs (macOS, every mainstream Linux,
WSL) and ships a layout that does not need configuration to be
useful — point it at a host and it tells you the story.

## Install

```bash
# Homebrew (macOS / Linux) — pulls Node as a dependency
brew install gtop

# npm (any host with Node ≥ 14)
npm install -g gtop

# Linux package managers
# Arch (AUR): yay -S gtop
# Nix: nix-env -iA nixpkgs.gtop

# Docker
docker run --rm -it --pid=host --net=host aksakalli/gtop

# verify
gtop --version    # 1.1.5

# launch
gtop
```

## What it does

- One-screen TUI: per-core CPU sparklines, memory + swap charts,
  network rx/tx chart, disk usage bars, sortable process table.
- Interactive process-table sort: `p`/`c`/`m` and `/` for filter.
- Cross-platform via Node + `os-utils` / `systeminformation` —
  works on macOS, mainstream Linux, WSL.
- Ships a Docker image (`aksakalli/gtop`) with `--pid=host
  --net=host` for "monitor the host from inside a sidecar."
- No config file required; sensible defaults out of the box.
- Tiny key surface (`q` to quit, arrows to navigate) — no manual
  page needed for first use.

## pew-related use cases

- **Quick remote triage**: `ssh host gtop` when an alert fires and
  you need CPU + memory + network + top processes on one screen
  without setting up Prometheus / Grafana for a one-off.
- **Live demo / screen-share** of a load test, where the audience
  needs to *see* CPU saturate and network ramp; `gtop` reads
  better on a projector than `htop` + `iftop` + `iostat` tiled
  across panes.
- **Container-host inspection** via the published Docker image
  with `--pid=host --net=host`, when the host doesn't have a
  package manager you trust to install monitoring tooling.
- **Sanity check for an AI-driven build / agent loop**: leave
  `gtop` open in a side tmux pane while an LLM coding agent
  ([opencode](../opencode/), [crush](../crush/), [aider](../aider/))
  runs a long task, so memory leaks or runaway subprocesses are
  visible immediately rather than after the OOM-killer fires.
- **Lightweight alternative to [bottom](../bottom/) /
  [zenith](../zenith/)** when you specifically want a Node-based
  install (e.g. on a host that already has Node but no Rust
  toolchain and no permission to add one).
