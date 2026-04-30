# impala

> **A keyboard-driven Wi-Fi management TUI
> built on `iwd` — list adapters, scan
> networks, connect / disconnect, manage known
> networks, and watch RSSI in real time
> without leaving the terminal** — a single
> Rust binary that wraps the `iwd` D-Bus API
> in a Ratatui interface for Linux desktops
> and headless boxes that have outgrown
> `iwctl`'s prompt. Pinned to **v0.7.4**
> ([LICENSE](https://github.com/pythops/impala/blob/main/LICENSE),
> GPL-3.0).

Source: <https://github.com/pythops/impala>

## TL;DR

`impala` is a Ratatui front-end for
[iwd](https://iwd.wiki.kernel.org/), the
modern Linux Wi-Fi daemon that replaces
`wpa_supplicant` on many distros (Arch,
Fedora workstation, Alpine, Void). The TUI
shows three vertical panes: **adapters**
(`wlan0` and friends, with state, mode, and
power), **known networks** (saved profiles
with auto-connect status), and **available
networks** (live scan with SSID, signal
strength bar, security type, frequency
band). Single-key actions cover the full
day-to-day flow: `s` to start a fresh scan,
`Enter` to connect (with a password prompt
modal for protected networks), `d` to
disconnect, `r` to remove a known network,
`Tab` to cycle panes. The signal-strength
column updates live as scans roll in, so you
can walk around a building and watch RSSI
move; for hotspot debugging there's an
**access-point mode toggle** that flips the
adapter to AP and lets you publish an SSID
straight from the TUI. Configuration is a
small TOML at `~/.config/impala/config.toml`
(colour theme, default scan interval, key
bindings); the binary itself is a single
~5 MB static musl build.

## Install

```bash
# Arch Linux (AUR)
yay -S impala

# Cargo (from crates.io)
cargo install impala

# from source
git clone https://github.com/pythops/impala
cd impala && cargo install --path .

# prebuilt static binaries (Linux x86_64 + aarch64, musl)
# https://github.com/pythops/impala/releases

# verify
impala --version    # 0.7.4
```

Requires `iwd` running as the Wi-Fi backend
(`systemctl enable --now iwd`) and the
invoking user in the `network` group (or
equivalent polkit rule) so D-Bus calls to
`net.connman.iwd` succeed.

## Basic usage

```bash
# launch the TUI
impala

# launch in access-point mode (adapter as AP)
impala --mode ap
```

In the TUI:

```text
Tab / Shift-Tab   cycle panes (Adapters / Known / Networks)
j / k             move selection
s                 start scan
Enter             connect to selected network
                  (password prompt opens for WPA/WPA2/WPA3)
d                 disconnect current connection
r                 remove selected known network
a                 toggle adapter power on / off
m                 switch adapter mode (Station / AP)
?                 help
q                 quit
```

## When to choose

- **Your distro uses `iwd` and you want a
  full-screen view of nearby networks +
  current RSSI without `iwctl`'s
  command-prompt loop** — `impala` is to
  `iwctl` what [`k9s`](../k9s/) is to
  `kubectl`: same backend, far less typing.
- **You manage Wi-Fi on a headless box over
  ssh** — single binary, no GUI deps, no
  D-Bus tray icon required; works inside
  `tmux` on a server you only ever reach
  through a terminal.
- **You want to watch signal-strength change
  in real time while debugging coverage** —
  the live scan column is the simplest "walk
  around with a laptop and see RSSI move"
  tool that doesn't need `wavemon` /
  `airodump-ng` (and avoids the
  attack-tooling baggage of the latter).

## When NOT to choose

- **Your distro uses NetworkManager**
  (Ubuntu desktop, Fedora server, Pop!_OS,
  Mint) — `impala` only speaks `iwd`'s D-Bus
  surface; use [`nmtui`](https://networkmanager.dev/)
  (ships with NetworkManager) or
  [`nmcli`](https://networkmanager.dev/) for
  CLI.
- **You need cellular / Bluetooth / VPN
  management in the same TUI** —
  NetworkManager front-ends cover the whole
  connectivity stack; `impala` is Wi-Fi only.
- **You need wireless packet capture or
  network-attack tooling** — out of scope
  by design (and out of scope for this zoo);
  `impala` is a connection manager, not a
  monitor / injection tool.

## Why it fits the zoo

Joins the "single-purpose system-management
TUI" cluster alongside
[`systemctl-tui`](../systemctl-tui/)
(`systemd` units),
[`oxker`](../oxker/) (Docker containers),
[`lazydocker`](../lazydocker/) (Docker, fuller
surface), [`k9s`](../k9s/) / [`kdash`](../kdash/)
(Kubernetes), [`bandwhich`](../bandwhich/)
(per-process bandwidth), and
[`sniffnet`](../sniffnet/) (network-traffic
TUI). `impala`'s specific gap is **a Wi-Fi
station / AP manager that targets the modern
`iwd` daemon directly**, which the zoo did
not previously cover — every other entry
either targets a different layer (process,
container, kube, traffic) or a different
backend (NetworkManager-only tools aren't
catalogued because the `nm*` tooling ships
with the daemon itself).

## Upstream pointers

- Repo: <https://github.com/pythops/impala>
- Release notes: <https://github.com/pythops/impala/releases>
- License: [GPL-3.0](https://github.com/pythops/impala/blob/main/LICENSE) (`LICENSE`)
- Maintainer: [@pythops](https://github.com/pythops)
- Backend: [iwd](https://iwd.wiki.kernel.org/) (Intel Wireless Daemon, Linux only)
