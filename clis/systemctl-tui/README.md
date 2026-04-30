# systemctl-tui

> **A fast TUI for browsing, starting, stopping,
> and tailing systemd units** — fuzzy-search the
> unit list, hit Enter to inspect status, and
> stream `journalctl` output for the focused
> unit in a side pane, all without retyping
> service names. Pinned to **v0.5.2**
> ([LICENSE](https://github.com/rgwood/systemctl-tui/blob/master/LICENSE),
> MIT).

Source: <https://github.com/rgwood/systemctl-tui>

## TL;DR

`systemctl-tui` is a single Rust binary that
wraps the `systemctl` and `journalctl`
commands behind a Ratatui interface. Launch it
and you get a left pane listing every unit
visible to the current user (or system, with
`--scope system`), filterable by a top-of-pane
fuzzy search, and a right pane showing the
selected unit's status block plus a live
journal tail. Single-key actions cover the
common verbs — `s` start, `S` stop, `r`
restart, `R` reload, `e` enable, `d` disable,
`/` search — and every action prints the
underlying `systemctl` invocation in a status
bar so you learn the command form by osmosis.
The journal pane streams via
`journalctl --follow -u <unit>` under the hood,
honors `j`/`k` scrolling, and pauses follow when
you scroll up. Works against the **user bus**
(`--user`, the default) or the **system bus**
(`--scope system`) and over **SSH** — the binary
is `~5 MB` static, so `scp systemctl-tui
host:/usr/local/bin/` and you have a TUI on a
box without compiling anything.

## Install

```bash
# Cargo
cargo install systemctl-tui --locked

# Arch Linux (AUR)
paru -S systemctl-tui

# prebuilt binaries (Linux x86_64 / aarch64)
# https://github.com/rgwood/systemctl-tui/releases

# verify
systemctl-tui --version    # systemctl-tui 0.5.2
```

## Basic usage

```bash
# user units (default scope)
systemctl-tui

# system-wide units (root or polkit-allowed)
sudo systemctl-tui --scope system

# launch with the search box pre-filled
systemctl-tui --search nginx

# limit unit types
systemctl-tui --unit-type service,socket
```

Inside the TUI: `/` filters, `Enter` focuses,
`s`/`S`/`r`/`R` runs the lifecycle verb,
`Tab` swaps focus between unit list and journal
pane, `q` quits.

## When to choose

- **You are debugging a service on a box and
  keep typing
  `systemctl status nginx; journalctl -fu nginx`**
  — systemctl-tui collapses both into one screen
  and lets you switch units without retyping.
- **You want a discoverable surface for
  `systemctl`** — fuzzy-searching all units +
  the printed command line for every action is
  a friendlier teaching tool than `man
  systemctl`.
- **You SSH into a Linux box from a laptop
  often** — small static binary, pure terminal,
  works fine on a slow link.

## When NOT to choose

- **You're not on systemd** (Alpine / OpenRC,
  macOS launchd, BSD rc.d, Windows services) —
  this binary literally shells out to
  `systemctl` and `journalctl`; use the native
  service manager's tools.
- **You want metric / health graphs per unit** —
  use [`bottom`](../bottom/), [`ctop`](../ctop/),
  or a Prometheus exporter; systemctl-tui shows
  state and logs, not CPU/memory time series.
- **You're writing a deploy script** — call
  `systemctl` and `journalctl` directly; this
  is an interactive tool.

## Why it fits the zoo

Belongs to the "tight TUI over a single Linux
subsystem" cluster
([`oxker`](../oxker/) for Docker,
[`lazydocker`](../lazydocker/) for Docker,
[`lazyjj`](../lazyjj/) for jj, `lazysql` for
databases). systemctl-tui's specific gap is
**user + system bus systemd unit management
with a built-in journal tail** — nothing else in
the catalog does that under one keymap.

## Upstream pointers

- Repo: <https://github.com/rgwood/systemctl-tui>
- Release notes: <https://github.com/rgwood/systemctl-tui/releases>
- License: [MIT](https://github.com/rgwood/systemctl-tui/blob/master/LICENSE) (`LICENSE`)
- Maintainer: [@rgwood](https://github.com/rgwood)
