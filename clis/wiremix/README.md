# wiremix

> **TUI mixer for PipeWire** — a single Rust binary that talks
> directly to the PipeWire daemon (no `pw-cli` shell-out, no
> JACK / PulseAudio shim) and renders an interactive mixer:
> per-stream volume + mute + balance, per-device default-sink /
> default-source selection, vu-meter visualisation, and a tabbed
> view across **Playback / Recording / Output Devices / Input
> Devices / Configuration**, all keyboard-driven and themable
> via TOML — pinned to **v0.10.0** (commit
> [`97b7ed1`](https://github.com/tsowell/wiremix/commit/97b7ed155440d1680f0c0d4c83e0fa178a2a9ec7),
> [LICENSE-MIT](https://github.com/tsowell/wiremix/blob/v0.10.0/LICENSE-MIT)
> + [LICENSE-APACHE](https://github.com/tsowell/wiremix/blob/v0.10.0/LICENSE-APACHE),
> dual-licensed `MIT OR Apache-2.0`).

Source: <https://github.com/tsowell/wiremix>

## TL;DR

PipeWire is the modern Linux audio + video router that has
displaced PulseAudio and JACK on most desktops since ~2022.
The official user-facing CLI is `pw-cli` / `pw-dump` — a
JSON-emitting introspection tool, not a mixer; the official
GUI is `qpwgraph` / `helvum` (graph editor) — overkill when
all you want to do is "lower the volume on this Firefox tab"
or "switch the default output to the headphones."

`pavucontrol` + `pulsemixer` cover the PulseAudio surface but
talk to PipeWire only via the `pipewire-pulse` compatibility
shim — they cannot see PipeWire-native streams that bypass
the Pulse layer (most modern pro-audio apps), and they miss
the per-node parameters (channel maps, sample rates, latency
quanta) that are only exposed on the native API.

`wiremix` is the right shape: a `pulsemixer`-style ncurses
mixer that speaks PipeWire natively. Five tabs (Playback,
Recording, Output Devices, Input Devices, Config), arrow-key
navigation, `+` / `-` to nudge volume, `m` to mute, `d` to
set default device, `Tab` to switch panels, `?` for help,
`q` to quit. Every change is sent over the PipeWire socket
and reflected back the same way `qpwgraph` would render it.

## Install

```bash
# Cargo (any Linux host with a Rust 1.74+ toolchain)
cargo install wiremix

# Arch Linux (AUR)
yay -S wiremix

# Pre-built x86_64 binary from upstream tags
curl -L https://github.com/tsowell/wiremix/releases/download/v0.10.0/wiremix-x86_64-unknown-linux-gnu.tar.gz | tar xz
```

Requires PipeWire 0.3.32+ at runtime (any modern distro:
Fedora 35+, Ubuntu 22.10+, Arch, NixOS unstable). The
`pipewire-devel` / `libpipewire-0.3-dev` headers are needed
at build time only, not at runtime.

## Example usage

```bash
# launch the TUI
wiremix

# start on a specific tab
wiremix --tab playback
wiremix --tab output-devices

# point at a non-default PipeWire socket
PIPEWIRE_REMOTE=/run/user/1000/pipewire-1 wiremix

# load a custom theme
wiremix --config ~/.config/wiremix/wiremix.toml
```

In-TUI keymap (subset):

- `←` / `→` adjust volume by 1 %; `+` / `-` by 5 %
- `m` toggle mute
- `d` make selected node the default sink/source
- `Tab` / `Shift+Tab` cycle the five tabs
- `Enter` expand a stream → per-channel volume + balance
- `?` help, `q` quit

## Why it matters

The PipeWire migration left a usability gap: for the first
two years there was *no* dedicated TUI mixer that spoke the
native API, only the Pulse-shim-via-`pavucontrol` /
`pulsemixer` workaround. `wiremix` closes that gap with one
self-contained ~5 MB binary that runs over SSH on a 80 × 24
terminal — the same shape that made `pulsemixer` the
reflexive answer for headless Linux audio for a decade.
Especially valuable for:

- A tiling-WM Linux laptop (Sway / Hyprland / i3 / river)
  with no GNOME / KDE control panel installed.
- A Raspberry Pi or NUC acting as a PipeWire audio sink
  reached only over SSH — `wiremix` is one keystroke away
  from "switch playback to the USB DAC."
- A pro-audio workstation where `pavucontrol` cannot see
  the Bitwig / Ardour / Reaper streams because they bypass
  the Pulse shim — `wiremix` sees them because it speaks
  the native protocol.
- A Wayland-only session where the `pavucontrol-qt` GUI
  costs a Qt6 runtime.

Pairs with [`bluetuith`](../bluetuith/) (sibling TUI for the
*Bluetooth connection* layer — `bluetuith` pairs the headset,
`wiremix` routes audio to it; together they cover the radio +
audio gap on a headless Linux box), with `qpwgraph` (graph
editor for *routing* PipeWire nodes — orthogonal: `qpwgraph`
draws the wires, `wiremix` sets the volumes), and with
`pw-top` (live latency / underrun monitor — observability
counterpart to `wiremix`'s control surface).

## License

`MIT OR Apache-2.0` (dual-licensed; downstream picks). See
[LICENSE-MIT](https://github.com/tsowell/wiremix/blob/v0.10.0/LICENSE-MIT)
and
[LICENSE-APACHE](https://github.com/tsowell/wiremix/blob/v0.10.0/LICENSE-APACHE)
in upstream.

## As of

2026-05-04. Upstream tag `v0.10.0` (latest tag on
`tsowell/wiremix`, released 2026-03-06). Linux + PipeWire
only; macOS / Windows / FreeBSD do not ship PipeWire (use
the platform-native mixer there).
