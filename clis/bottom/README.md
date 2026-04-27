# bottom

> **Cross-platform graphical system monitor in the terminal** — a
> single Rust binary (`btm`) that opens a `htop`-class TUI with
> *graphs* (CPU per-core, memory, network, temperature, disk IO,
> battery) drawn as braille-character sparklines, a sortable
> filterable process table with tree mode, kill / nice signals
> bound to one keystroke, and basket-config for layouts so the
> dashboard you want is the dashboard you boot. Pinned to
> **0.12.3** (commit
> `4a8f1d3a8e26fbb09d8cdf8e76eaf81b50c7d5b1`,
> [LICENSE](https://github.com/ClementTsang/bottom/blob/main/LICENSE),
> MIT).

Source: <https://github.com/ClementTsang/bottom>

## TL;DR

`bottom` (binary name `btm`) is what you reach for when `htop`'s
"a wall of numbers" stops scaling — when you want to *see* CPU
spike across the last 60 seconds, watch memory grow as a curve,
correlate a network burst against a process spawn — without
opening Grafana. `btm` runs anywhere `htop` runs (Linux, macOS,
FreeBSD) plus Windows (where `htop` does not), draws braille-
character graphs that look passable on any terminal, and binds
the operations you actually do (kill a runaway process, sort by
CPU then by memory, filter by name, expand a process tree) to
single keystrokes. Default layout is six widgets — CPU graph (per-
core or aggregated), memory + swap graph, network graph
(rx / tx with cumulative + rate), temperature panel, disk IO
graph, process table — and `e` toggles widget-expand to fullscreen
the focused widget. Process table sorts on column header (`P`
CPU%, `M` mem%, `N` PID, `T` time alive, `c` count when grouped),
filters live with `/`, and kill is `dd` (vim muscle memory) →
signal picker → enter. `btm --basic` collapses the graphs into
plain numeric panels (the `htop` look) for low-bandwidth SSH.

## Install

```bash
# Homebrew (macOS / Linux)
brew install bottom

# Cargo
cargo install --locked bottom

# Linux package managers
# Arch: pacman -S bottom
# Nix: nix-env -iA nixpkgs.bottom
# Fedora: dnf copr enable atim/bottom && dnf install bottom
# Snap: snap install bottom
# Debian / Ubuntu: download .deb from releases page

# Windows
# scoop install bottom
# choco install bottom
# winget install bottom

# from a release tarball (any OS)
curl -Lo bottom.tar.gz "https://github.com/ClementTsang/bottom/releases/latest/download/bottom_aarch64-apple-darwin.tar.gz"
tar xf bottom.tar.gz btm
sudo install btm /usr/local/bin/

# verify
btm --version    # 0.12.3
```

Zero config required — `btm` runs with defaults out of the box.
For custom layouts (drop the temperature widget, add a per-core
CPU pane, change the refresh rate to 250 ms): `btm --generate_config
> ~/.config/bottom/bottom.toml` dumps the full default config as
TOML you can edit.

## License

MIT — see [LICENSE](https://github.com/ClementTsang/bottom/blob/main/LICENSE).
Permissive, no attribution required for binaries; redistribute
freely.

## Hot keybinds

These are the ones that pay back in the first session:

- `?` — help overlay (every keybind in one screen, dismissed with
  any key)
- `q` / `Ctrl+C` — quit
- `Tab` / `Shift+Tab` — cycle focus across widgets; arrow keys /
  `h` `j` `k` `l` move between widgets in 2D
- `e` — expand focused widget to fullscreen; press again to
  collapse back to the layout
- `f` — freeze / unfreeze the display (stop sampling, useful for
  reading a transient state without it scrolling away)
- **Process table:** `dd` — kill (signal picker appears, default
  `SIGTERM`); `/` — filter (regex by default, prefix with `\` for
  literal); `Ctrl+f` — search (highlights matches without filtering
  out non-matches); `s` — sort by column; `S` — reverse sort; `t`
  — toggle tree mode (process hierarchy with parent / child
  indentation); `g` — toggle grouped mode (collapse same-name
  processes into one row with child count); `P` / `M` / `N` / `T`
  — sort by CPU% / mem% / PID / time
- **CPU graph:** `+` / `-` — zoom in / out on the time axis;
  `=` — reset zoom; `Ctrl+arrow` — scroll the time window
- **Network graph:** same time-axis zoom + scroll; `k` toggles
  unit (bits / bytes); `T` toggles cumulative total visibility
- **Disk + temperature panels:** sort columns same as process
  table (`s` / column letter)
- `Ctrl+r` — reload config (re-reads `~/.config/bottom/bottom.toml`
  without restart)
- `b` — toggle battery panel (laptops); `B` — toggle bold text
- `Esc` — close any popup / modal (signal picker, help overlay,
  filter input)

## Why use it

Three things `bottom` does that `htop` / `top` do not, that pay
back the switching cost:

1. **Graphs make rates visible.** `htop` shows you "CPU is at 87%
   right now"; `btm` shows you "CPU spiked from 12% to 87% over
   the last 8 seconds and is now plateauing" — the curve answers
   the next question (is this a runaway, a workload, or a brief
   spike) before you have to think about it. The 60-second time
   window is configurable via `--default_time_value` and zoomable
   live with `+` / `-`. The braille rendering means the graphs
   look fine in any terminal, no graphics protocol needed.
2. **Cross-platform including Windows.** `btm` runs natively on
   Windows (PowerShell, Windows Terminal, cmd.exe) where `htop`
   does not — relevant when you SSH into a Windows host or when
   your dev box is Windows. Same keybinds on macOS / Linux /
   Windows / FreeBSD, so the muscle memory transfers.
3. **Layouts as TOML config.** The default six-widget layout is
   one of many — `~/.config/bottom/bottom.toml`'s `[[row]]` /
   `[[row.child]]` blocks declare a tree of widgets with width
   ratios, so you can build a "CPU per-core full width on top,
   process table full width on bottom" layout for one server, a
   "network graph + process table only" layout for another, and
   switch with `--config /path/to/layout.toml`. `htop`'s meters
   are configurable but layout is fixed; `btm`'s layout is fully
   declarative.

For an LLM-CLI workflow where a long-running agent (e.g. a fine-
tune, a batch eval, a training run) needs to be watched for
"is the GPU saturated, is the memory leaking, did the process
just die" without opening a separate dashboard, `btm` running in
a `zellij` / `tmux` pane gives you the answer at a glance.

## Vs Already Cataloged

- **Vs `htop` (not cataloged):** the closest peer. `htop` is on
  every Linux box already, has a smaller binary, and is well-
  understood. Pick `htop` for "I just want a process list on a
  random server"; pick `btm` when graphs over time + Windows
  support + declarative layouts matter.
- **Vs `glances` (not cataloged):** Python-based, broader plugin
  ecosystem (Docker stats, sensors, GPU panels via plugins),
  slower refresh rate due to Python overhead, more `htop`-shaped
  layout. Pick `glances` for the plugin breadth, `btm` for the
  graphs + speed + cross-platform consistency.
- **Vs [`zellij`](../zellij/):** orthogonal — `zellij` is a
  terminal multiplexer, `btm` is a system monitor that runs *in*
  a pane. A common dev workspace: editor in pane 1, REPL in
  pane 2, `btm` in a small pane 3 watching the machine.
- **Vs [`atuin`](../atuin/):** orthogonal — `atuin` is shell
  history, `btm` is a process / resource monitor. They share the
  "rich TUI in a pane" niche but solve different problems.
- **Vs [`yazi`](../yazi/):** orthogonal — `yazi` is a file
  manager, `btm` is a system monitor.

## Caveats

- **Process names on macOS are sometimes truncated** (the kernel
  exposes a 16-char name in some APIs); the full path / argv is
  available in expanded view (`e` on the process widget) but the
  default column is short. Linux / Windows do not have this
  limitation.
- **GPU panels are limited.** `btm` 0.12 supports NVIDIA GPUs via
  NVML on Linux / Windows (`--enable_gpu` or config flag); AMD
  and Apple Silicon GPUs are not first-class. For ML workloads
  on a Mac, pair with `asitop` or `mactop` for GPU stats.
- **Sampling is at fixed intervals** (default 1 s); a sub-second
  CPU spike between samples is invisible. `--rate 250` samples
  every 250 ms (CPU cost goes up correspondingly). Not a
  replacement for `perf` / `bpftrace` for sub-second observation.
- **`dd` to kill is two keystrokes** and the signal picker is
  small — easy to muscle-memory through on the wrong row if you
  did not just verify the cursor position. Bottom prints the
  selected row name in the picker title; read it before pressing
  enter.
- **Battery panel reads vendor-specific APIs.** Works on most
  laptops (Apple, ThinkPad, XPS, modern Surface) but some BIOSes
  expose battery state in non-standard ways and the panel shows
  blanks. Toggle off with `b` if it is noise.
- **Config schema has had breaking changes** between 0.x minor
  versions — a `bottom.toml` from 0.9 may not parse cleanly under
  0.12. Re-run `btm --generate_config` after a major upgrade and
  port your customizations.
