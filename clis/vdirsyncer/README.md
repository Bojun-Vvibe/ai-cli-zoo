# vdirsyncer

> **CalDAV / CardDAV ↔ local-filesystem sync daemon** — bidirectional
> synchronizer that turns remote calendar/contact servers into plain
> `.ics` and `.vcf` files in a directory tree, so command-line tools
> like [`khal`](../khal/), [`khard`](../khard/), and `todoman` can
> operate offline. Pinned to **v0.19.3** (released 2024-04-13,
> [LICENSE.txt](https://github.com/pimutils/vdirsyncer/blob/main/LICENSE.txt),
> 3-clause BSD).

Source: <https://github.com/pimutils/vdirsyncer>

## TL;DR

`vdirsyncer` is a one-shot sync command (typically wired to
`systemd --user` or `cron` / `launchd`) that pairs a *remote*
storage (a CalDAV or CardDAV server: Nextcloud, Radicale,
Fastmail, Google, iCloud, etc.) with a *local* storage (a
`vdir`-format directory of one `.ics` or `.vcf` file per item).
After `vdirsyncer discover` walks the server and `vdirsyncer
sync` runs the bidirectional merge, your calendars and address
books exist as plain text files on disk that any editor, grep,
or git workflow can read — and any TUI in the `pimutils` family
(`khal` for calendar, `khard` for contacts, `todoman` for VTODO
tasks) can render and edit, with changes flowing back to the
server on the next sync. It is the canonical "make my CalDAV
server look like a folder of files" tool for the Unix CLI PIM
stack.

## Why pick it over alternatives

Pick `vdirsyncer` when you want CLI / TUI access to calendars and
contacts that live on a CalDAV / CardDAV server and you accept
the "sync on a timer" model (it is not a live IMAP-IDLE-style
push). Compared to native clients (Apple Calendar, Thunderbird,
DAVx⁵): those are GUI apps and do not produce a directory you
can grep or version-control. Compared to **Etebase / EteSync**:
those are end-to-end-encrypted *server* alternatives — you would
still need vdirsyncer (or its etesync-aware cousin) to land the
data on disk. Compared to scripting `curl` against the CalDAV
endpoint yourself: vdirsyncer handles the WebDAV ETag / CTag
dance, conflict detection, and the discovery of collections,
which is the bulk of the work. Skip vdirsyncer if you only use
GUI calendar/contact apps, if your data lives in Google
Calendar and you are happy with the web UI, or if you need
real-time push notifications — vdirsyncer is poll-based by
design.

## Install

```bash
# macOS / Linux
brew install vdirsyncer
# or
pipx install vdirsyncer

# Debian / Ubuntu
sudo apt install vdirsyncer

# verify
vdirsyncer --version    # 0.19.3
```

Quick start (Nextcloud-style CalDAV → local vdir):

```bash
# minimal config at ~/.config/vdirsyncer/config
cat > ~/.config/vdirsyncer/config <<'EOF'
[general]
status_path = "~/.local/share/vdirsyncer/status/"

[pair my_calendars]
a = "my_calendars_local"
b = "my_calendars_remote"
collections = ["from a", "from b"]

[storage my_calendars_local]
type = "filesystem"
path = "~/.calendars/"
fileext = ".ics"

[storage my_calendars_remote]
type = "caldav"
url = "https://cloud.example.com/remote.php/dav/"
username = "alice"
password.fetch = ["command", "secret-tool", "lookup", "service", "vdirsyncer"]
EOF

# one-time discovery, then routine sync
vdirsyncer discover
vdirsyncer sync

# wire to a launchd / systemd timer for periodic sync
```
