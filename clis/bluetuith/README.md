# bluetuith

> **TUI-based Bluetooth manager for Linux** — a single Go
> binary that talks to BlueZ over D-Bus and renders an
> interactive terminal interface for everything `bluetoothctl`
> can do (and several things it cannot): scan for devices,
> pair / trust / connect / disconnect, browse paired devices,
> manage adapters, send and receive files via OBEX OPP, control
> connected media via AVRCP, and toggle adapter properties —
> pinned to **v0.2.6** (commit tag `v0.2.6`,
> [LICENSE](https://github.com/darkhz/bluetuith/blob/master/LICENSE),
> MIT).

Source: <https://github.com/darkhz/bluetuith>

## TL;DR

`bluetoothctl` is the official BlueZ CLI; it is a REPL that
prints unsorted event lines and accepts colon-separated MAC
addresses by hand. `bluetuith` is the same set of operations
behind a `tview` TUI: a sorted list of devices on the left
(connected → paired → unpaired, with signal strength + class
icons), a properties pane on the right, single-key actions
(`c` connect, `d` disconnect, `p` pair, `t` trust, `r` remove,
`s` send file, `a` audio profile, `o` adapter), a status line,
and a help overlay (`?`). All of this is one ~10 MB Go binary
with no Python / Qt / GTK runtime — the right shape for a
headless server, an Alpine container with a USB Bluetooth
adapter, a chromebook in dev mode, or a Raspberry Pi running
nothing else.

The killer property is the **OBEX OPP file transfer**: sending
a file to a phone over Bluetooth from `bluetoothctl` requires
running `obexctl` separately, knowing the phone's OBEX channel,
and copy-pasting the MAC; in `bluetuith` it is one keystroke
on the device row plus a path in a file picker, with progress
in the status line.

## Install

```bash
# Go install (any platform with a Go toolchain + Linux target)
go install github.com/darkhz/bluetuith@latest

# Arch Linux (AUR)
yay -S bluetuith

# Pre-built binary
# (download from upstream releases page)
```

Requires BlueZ 5.x and a running `bluetoothd`; on systemd hosts
that is `systemctl enable --now bluetooth`. Works against any
adapter BlueZ supports (USB dongles, built-in laptop chips,
Raspberry Pi 4/5 onboard radios). OBEX file transfer requires
`obexd` (`apt install obexd` / `pacman -S bluez-obex`) running
on the user session bus.

## Example usage

```bash
# launch the TUI
bluetuith

# choose a non-default adapter on a multi-radio host
bluetuith --adapter hci1

# disable the receive-files prompt (headless audio sink use)
bluetuith --receive-dir ""
```

In-TUI keymap (subset):

- `Tab` cycle adapter list
- `s` start scan, `S` stop scan
- `c` connect, `d` disconnect
- `p` pair, `t` trust, `r` remove
- `Ctrl+S` send file (OBEX OPP)
- `?` help overlay
- `q` quit

## Why it matters

The Linux Bluetooth UX has been "open the GNOME / KDE settings
panel" or "hand-roll `bluetoothctl` commands" for a decade.
`bluetuith` is the missing middle: full Bluetooth control on a
host with no display server, full keyboard navigation, no
per-distro settings panel to learn, and the same workflow on
every desktop and headless host. Especially valuable for:

- A laptop running a tiling WM (Sway, Hyprland, i3) where the
  GNOME control centre is not installed.
- A Raspberry Pi acting as a Bluetooth audio sink — `bluetuith`
  over SSH lets you pair the adapter to a phone in 30 seconds
  without exporting an X session.
- An air-gapped server with a USB Bluetooth dongle for a one-
  off file transfer to a technician's phone, no GUI needed.

Pairs with `bluez-tools` (`bt-adapter`, `bt-device`, `bt-obex`
— scriptable single-shot CLIs for unattended automation, where
`bluetuith` is the interactive surface) and with
[`impala`](../impala/) (sibling Wi-Fi TUI by the same shape;
together they cover the radio-management gap on headless
Linux). Orthogonal to `pulsemixer` / `pavucontrol` /
`wiremix` (those are PulseAudio / PipeWire mixers — manage
*audio routing* once the Bluetooth audio device is connected;
`bluetuith` manages *the Bluetooth connection itself*).

## License

MIT. See
[LICENSE](https://github.com/darkhz/bluetuith/blob/master/LICENSE)
in upstream.

## As of

2026-05-04. Upstream tag `v0.2.6` (latest GitHub release on
`darkhz/bluetuith` as of this snapshot). The BlueZ D-Bus API is
stable; bluetuith's keymap and flag set may evolve — re-check
the upstream README before pinning in scripted contexts.
