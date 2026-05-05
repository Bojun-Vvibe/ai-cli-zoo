# musikcube

> **Cross-platform terminal-native music player + library
> server in one binary** — a C++ ncurses TUI that indexes a
> local music library (FLAC / MP3 / OGG / Opus / WAV / DSD /
> ALAC / AAC / WMA via plugins), plays gapless with ReplayGain
> and crossfade, exposes a built-in HTTP / WebSocket server
> for remote control, and ships matching mobile / desktop
> remotes that drive the same daemon — so the same `musikcube`
> process is your headless audio server, your terminal player,
> and your phone's "play / pause / queue" backend. Pinned to
> **v3.0.5** (commit `9d0b7939642dacd7411e67e55cbb8e997e3c99cb`,
> [LICENSE.txt](https://github.com/clangen/musikcube/blob/master/LICENSE.txt),
> SPDX: `BSD-3-Clause`).

Source: <https://github.com/clangen/musikcube>

## TL;DR

`musikcube` is a single C++ TUI binary that reads `~/Music`
(or any path) into a SQLite-backed library, classifies tracks
by tag (artist / album / genre / year / playcount), and plays
them through a pluggable audio backend (`pulseaudio`, `pipewire`,
`alsa`, `coreaudio`, `wasapi`, `directsound`) with gapless
playback, crossfade, ReplayGain, and a 10-band equaliser.
Inside the same process it runs an authenticated HTTP +
WebSocket server (`musikcube server`) that the official
Android / iOS / macOS / Windows `musikdroid` / `musikcube`
remote apps connect to — so a Raspberry Pi running headless
`musikcube` on a stereo amp becomes a phone-controlled music
server with no separate Logitech-Media-Server-style stack.
The TUI itself is mouse + keyboard navigable, supports themes,
sub-second seek, queue editing, and a SHOUTcast-style streaming
input plugin for internet radio.

## Install

```bash
# macOS
brew install musikcube

# Debian / Ubuntu
# https://github.com/clangen/musikcube/releases/tag/3.0.5  (.deb)
sudo dpkg -i musikcube_*_amd64.deb

# Arch
yay -S musikcube

# Build from source (Linux / *BSD)
git clone https://github.com/clangen/musikcube
cmake -B build && cmake --build build -j

# verify
musikcube --version    # musikcube 3.0.5
```

First launch: `s` opens settings, `Add new path…` points at
your music root, library indexes in the background.

## License

BSD-3-Clause — see
[LICENSE.txt](https://github.com/clangen/musikcube/blob/master/LICENSE.txt).
Permissive: redistribute binaries and sources freely, retain
the copyright notice and the three-clause disclaimer; no
copyleft on derivative works.

## Representative Commands

```bash
# 1. interactive TUI (default)
musikcube
# arrow keys: navigate; enter: play; space: pause; n/p: next/prev
# q: enqueue; v: visualizer; t: theme; s: settings

# 2. run as a headless library server (no TUI)
musikcube --server
# listens on ws://0.0.0.0:7905 (metadata) + http://0.0.0.0:7906 (audio)

# 3. systemd unit for a Pi / NAS running 24/7
sudo systemctl --user enable --now musikcube.service

# 4. connect a phone: install musikdroid (F-Droid / Play),
#    enter host + port + password set in musikcube settings

# 5. add a streaming radio source
# inside TUI: s → Plugins → SHOUTcast → enter URL, save

# 6. rescan library after adding files
# inside TUI: s → Library → Rescan, or restart server
```

## Why It Matters

Most "play my own music collection" answers in 2026 are either
heavy (Plex, Jellyfin, Navidrome — all require a web UI, often
a separate frontend, sometimes a transcoder) or terminal-only
with no remote story (`cmus`, `ncmpcpp` over MPD, `mpv`).
`musikcube` is the one tool in the middle: a TUI you can drive
locally over SSH on a headless box, a server protocol the
official mobile apps speak natively (no DLNA / Sonos / Subsonic
shim), and a single binary that plays the same library three
different ways (TUI on the box, mobile app on the couch, web
in a pinch). Pairs with [`spotify-tui`](../spotify-tui/) /
[`ncspot`](../ncspot/) (those are Spotify front-ends,
`musikcube` is for files you own) and
[`termusic`](../termusic/) (a smaller Rust TUI player without
the server / mobile-remote story). Reach for `musikcube` when
you want **one process that is both the playback engine on a
headless host and the backend for a phone remote**, indexing a
file-based library you own — typical setup is a Pi 4 or NAS
with a USB DAC, `musikcube --server` under systemd, and a
phone app to queue albums from across the house. The killer
property is **first-class mobile remotes for a self-hosted
file library** without standing up a Subsonic-compatible
server next to your player; the same binary does both jobs.
