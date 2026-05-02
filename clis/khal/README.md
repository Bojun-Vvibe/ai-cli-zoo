# khal

> **Standards-based CLI / TUI calendar built on top of CalDAV +
> `vdirsyncer`** — a Python program that reads and writes plain
> iCalendar (`.ics`) files in a local directory tree and lets
> `vdirsyncer` round-trip them to any CalDAV server (Nextcloud,
> Radicale, Baikal, Sogo, Fastmail, Posteo, mailbox.org, generic
> WebDAV, or a Google account via OAuth). Pinned to **v0.11.3**
> (released 2024-02-12,
> [COPYING](https://github.com/pimutils/khal/blob/master/COPYING),
> MIT (Expat)).

Source: <https://github.com/pimutils/khal>

## TL;DR

`khal` is the answer when you want a calendar that lives next to
your shell and your text editor — `khal new tomorrow 14:00 1h
"sync with N"` adds an event without leaving the terminal,
`khal calendar` paints a month grid in the current pane, `ikhal`
opens an interactive `urwid` TUI with day / week / month views,
search, and edit-in-`$EDITOR` for full iCalendar fields. Storage
is one `.ics` file per event under
`~/.local/share/vdirs/<collection>/`, so backups, version
control, `grep`, and `rsync` all work without a special export
step. Multi-calendar support is per-collection (each maps to
one CalDAV calendar), with per-calendar colour, default
calendar, read-only flag, and a `priority` for conflict
resolution. Pairs naturally with [`vdirsyncer`](https://github.com/pimutils/vdirsyncer)
(the upstream sync engine — same `pimutils` org), with
[`khard`](https://github.com/lucc/khard) (the matching CLI
address book over CardDAV), and with
[`mutt`](https://gitlab.com/muttmua/mutt) /
[`neomutt`](https://github.com/neomutt/neomutt) /
[`aerc`](https://git.sr.ht/~rjarry/aerc) for a fully
terminal-native PIM stack.

## Install

```bash
# Homebrew (macOS / Linux)
brew install khal vdirsyncer

# pipx (any OS with Python 3.10+)
pipx install khal vdirsyncer

# Linux package managers
# Debian / Ubuntu: apt install khal vdirsyncer
# Fedora: dnf install khal vdirsyncer
# Arch: pacman -S khal vdirsyncer

# Configure (one-shot interactive wizard writes ~/.config/khal/config)
khal configure
```

## When to choose khal

- Your calendar source-of-truth is a CalDAV server you already
  run (Nextcloud, Radicale, Baikal, Sogo) or a hosted account
  that speaks CalDAV (Fastmail, Posteo, mailbox.org, iCloud
  with app-specific password).
- You live in a terminal and want event create / edit / search
  / RSVP without context-switching to a browser or GUI app.
- You want events stored as plain `.ics` files so they
  back up, diff, and `grep` like any other text artifact.
- You are already on the `pimutils` stack (`vdirsyncer` +
  `khard`) and want the calendar piece to match.

## When to pick something else

- You need a polished GUI calendar with drag-to-resize and
  shared-calendar invitations from a UI — use Thunderbird +
  Lightning, Apple Calendar, or the Nextcloud web UI.
- Your calendar lives only in Google Calendar with no CalDAV
  bridge — `gcalcli` talks to the Google API directly without
  a `vdirsyncer` round-trip.
- You want a single all-in-one PIM TUI that bundles mail +
  calendar + contacts in one process — see `aerc` (mail-first,
  calendar via plugins) or pull in the full `pimutils` set.

## Caveats

- `khal` itself does *not* talk CalDAV — `vdirsyncer` does the
  sync, `khal` reads/writes the local store. Set up
  `vdirsyncer` first; run `vdirsyncer sync` before/after
  `khal` sessions (cron / systemd timer / launchd).
- iCloud requires an app-specific password (Apple ID > Sign-In
  & Security > App-Specific Passwords); two-factor auth on the
  main Apple ID password will *not* work over CalDAV.
- Recurring event editing in `ikhal` modifies the master event
  by default; use `--instance` semantics from the upstream
  iCalendar spec for single-occurrence overrides.
