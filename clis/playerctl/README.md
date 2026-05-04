# playerctl

> **MPRIS D-Bus media-player remote control** — a single C
> binary (with `libplayerctl` as a sibling library) that
> speaks the MPRIS2 D-Bus interface to *any* media player
> on the system that registers an `org.mpris.MediaPlayer2.*`
> name (mpv, VLC, Rhythmbox, web browsers playing audio,
> Spotify, cmus, mpd-via-`mpDris2`, [`kew`](../kew/),
> [`spotify-player`](../spotify-player/), Audacious, …) and
> exposes seven verbs (`play`, `pause`, `play-pause`, `stop`,
> `next`, `previous`, `position`), three queries (`status`,
> `metadata`, `volume`), one introspection mode (`-l` lists
> active players), and a long-running `--follow` mode that
> emits one event per state change to stdout for status-bar
> and scripting consumers — pinned to **v2.4.1** (commit
> [`e5304e9d`](https://github.com/altdesktop/playerctl/commit/e5304e9dc9a0c0c32b3689c3f141cf266d27f59c),
> [COPYING](https://github.com/altdesktop/playerctl/blob/v2.4.1/COPYING),
> LGPL-3.0-or-later).

Source: <https://github.com/altdesktop/playerctl>

## TL;DR

The Linux desktop has one shared media-control protocol —
MPRIS2 over D-Bus — and ~30 unrelated media players
implement it. `playerctl` is the one CLI that bridges them:
`playerctl play-pause` toggles whichever player is currently
active, `playerctl --player=mpv next` targets a specific
player by name, and `playerctl --all-players pause` stops
every player on the bus when you walk away. The same binary
that desktop environments shell out to for the keyboard
media keys is the binary you wire into a `sxhkd` /
`Hyprland` / `Sway` / `i3` / `bspwm` config when the desktop
environment is not present.

The killer property is **`metadata --format` templating**:
`playerctl metadata --format "{{ artist }} — {{ title }}
[{{ duration(position) }}/{{ duration(mpris:length) }}]"`
prints a single line of formatted "now playing" text built
from any MPRIS field, with a small built-in DSL of
template helpers (`duration()` for nanoseconds → mm:ss,
`emoji(status)` for ▶/⏸/⏹, `markup_escape()` for safe Pango
output) — the exact shape `waybar` / `polybar` / `i3blocks` /
`tmux` `status-right` need to embed a status-bar widget
without parsing the JSON `metadata` output through `jq`. Pair
with `--follow` and the status bar updates in O(events) not
O(polls).

## Install

```bash
# Arch Linux (extra)
sudo pacman -S playerctl

# Debian / Ubuntu
sudo apt install playerctl

# Fedora
sudo dnf install playerctl

# Homebrew on Linux (macOS lacks D-Bus by default — see Caveats)
brew install playerctl

# Build from source (autotools / meson)
git clone --branch v2.4.1 https://github.com/altdesktop/playerctl
cd playerctl
meson setup build && ninja -C build
sudo ninja -C build install

# verify
playerctl --version    # v2.4.1
```

Requires a running D-Bus session bus (every modern desktop
session has one) and at least one MPRIS2-compliant player.
`libplayerctl` is the sibling library for GObject-introspected
language bindings (Python via `pygobject`, JavaScript via
`gjs`, Vala) — install `libplayerctl-dev` / `playerctl-devel`
for those.

## Example usage

```bash
# global media-key bindings (sxhkd / hypr / sway / i3 example)
XF86AudioPlay  -> playerctl play-pause
XF86AudioNext  -> playerctl next
XF86AudioPrev  -> playerctl previous
XF86AudioStop  -> playerctl stop

# stop every player on the bus (the "I'm leaving" hotkey)
playerctl --all-players pause

# target a specific player by name (introspect with -l)
playerctl -l                                  # mpv vlc spotify firefox
playerctl --player=mpv volume 0.5
playerctl --player=mpv volume 0.05+           # nudge up
playerctl --player=spotify next

# now-playing line for status bar
playerctl metadata --format "{{ artist }} — {{ title }}"

# follow mode: print one line per state change (waybar / polybar / tmux)
playerctl --follow metadata --format \
  "{{ emoji(status) }} {{ artist }} — {{ title }}"

# absolute and relative seek
playerctl position 30                          # jump to 30 s
playerctl position 10+                         # +10 s
playerctl position 10-                         # -10 s

# JSON metadata for jq-shaped pipelines
playerctl metadata --format \
  '{"player":"{{ playerName }}","artist":"{{ artist }}","title":"{{ title }}"}'
```

Common flags:

- `--player=NAME` target one player (omit to use first active)
- `--all-players` apply to every MPRIS player on the bus
- `--ignore-player=NAME` exclude one player from `--all-players`
- `-l` / `--list-all` list MPRIS bus names currently registered
- `-f FMT` / `--format` template DSL for `metadata` / `status`
- `-F` / `--follow` long-running, emit one event per state change
- `-s` / `--shuffle` toggle / set shuffle (when player supports it)
- `-r` / `--loop` toggle / set loop mode (Track / Playlist / None)
- `--version` / `--help`

## Why it matters

- **One CLI for ~30 players.** The alternative is per-player
  bespoke control (`mpc` for mpd, `dbus-send` to mpv's IPC
  socket, Spotify's web API, browser-extension recipes) —
  playerctl collapses that into seven verbs that work across
  the entire MPRIS surface.
- **The status-bar template DSL is the killer feature.** A
  status bar (`waybar`, `polybar`, `i3blocks`, `tmux
  status-right`) needs a one-line "now playing" string that
  updates on event, not on a 1 s poll. `playerctl --follow
  metadata --format "{{ emoji(status) }} {{ artist }} — {{
  title }}"` is one line in the bar config and the bar
  receives one update per actual state change.
- **`--all-players` is the "I'm leaving" hotkey.** Bind it
  to `Super+Esc` and walking away from the keyboard pauses
  every player at once — the alternative is identifying
  which player is producing audio (impossible without
  introspection) and pausing it specifically.
- **Volume control across players** — `playerctl
  --player=mpv volume 0.05+` raises mpv's *application*
  volume on the player side, distinct from the system /
  PipeWire stream volume that [`wiremix`](../wiremix/) /
  `pavucontrol` control. The two compose: per-player volume
  via playerctl, per-stream sink routing via wiremix.
- **GObject Introspection bindings** mean a Python /
  JavaScript / Vala script can subscribe to the same MPRIS
  events without re-implementing the protocol — useful for
  small custom widgets and notification bridges.

## Vs Already Cataloged

- **Vs [`kew`](../kew/) / [`spotify-player`](../spotify-player/) /
  [`spotify-tui`](../spotify-tui/) / [`ncmpcpp`](../ncmpcpp/):**
  orthogonal — those are *players* with their own TUI;
  playerctl is the cross-player *remote*. They compose:
  kew plays the music, playerctl's `play-pause` keystroke
  toggles it from anywhere on the desktop, the status bar
  uses `playerctl metadata --format` to display what's
  playing.
- **Vs [`wiremix`](../wiremix/) / `pavucontrol` / `pulsemixer`:**
  orthogonal axes — those control *system audio*
  (per-stream / per-device volume on PipeWire / Pulse);
  playerctl controls *player state* (pause / next /
  metadata) at the application level. Both are needed —
  changing volume in a status bar usually wants
  per-stream `wiremix` semantics, while a play-pause
  hotkey wants playerctl semantics.
- **Vs `dbus-send` / `gdbus call --session --dest
  org.mpris.MediaPlayer2.<player> ...`:** playerctl is
  exactly the syntactic-sugar layer on top of those —
  same protocol, much terser invocation. Use raw
  `dbus-send` / `busctl` only when you need an MPRIS
  field playerctl does not template (rare).
- **Vs `mpc` (mpd client) / `cmus-remote` / VLC's HTTP
  remote / Spotify's web API:** playerctl works against
  any of those when the player exposes MPRIS — for mpd
  via the `mpDris2` bridge — and uses one keymap across
  all of them. The native per-player CLIs remain the
  right pick when you need a player-specific feature
  (mpc's playlist / mpd-server commands have no MPRIS
  equivalent).

## License

LGPL-3.0-or-later — see
[COPYING](https://github.com/altdesktop/playerctl/blob/v2.4.1/COPYING).
The `playerctl` binary is LGPL — using it from a closed-source
shell script / status-bar config is unrestricted. Linking
`libplayerctl` into a closed-source application is permitted
under LGPL terms (dynamic linking + replaceability).

## Caveats

- **Linux-first; macOS does not ship D-Bus by default.**
  macOS users can install D-Bus via Homebrew and run
  playerctl, but the macOS players (Apple Music, Spotify
  on macOS) do not expose MPRIS — they use AppleScript /
  MediaRemote instead. For macOS media control use
  `nowplaying-cli` or AppleScript-based tools; playerctl
  is the Linux answer.
- **Last upstream release was 2021-09-21 (v2.4.1).** The
  MPRIS2 spec is stable and playerctl's surface has not
  meaningfully needed updates since; this is "stable not
  stalled." Active issues exist for niche bugs but the
  daily-driver workflows are unaffected.
- **`--follow` mode requires the player to emit
  `PropertiesChanged` on the bus.** Most well-behaved
  players do, a few only update on direct query — if a
  status bar shows stale data with one specific player,
  fall back to `watch -n 1 playerctl metadata` for that
  player.
- **Web browser MPRIS support is per-browser-version.**
  Firefox enables MPRIS by default; Chromium/Chrome
  require `--enable-features=GlobalMediaControlsForChromeOS`
  on some builds. If `playerctl -l` does not show the
  browser, check `chrome://flags` / `about:config`.
- **Player-name disambiguation when multiple instances
  run.** Two `mpv` processes appear as `mpv.instance1234`
  and `mpv.instance5678` — playerctl picks the
  most-recently-active by default, but a status bar
  pinning one specific instance needs the full
  bus-name suffix.
- **No write-side authentication.** Any process on the
  user's session bus can send `playerctl pause`. This is
  the MPRIS spec, not a playerctl bug; relevant only on
  multi-tenant Unix shells (rare on a personal Linux
  desktop).

## As of

2026-05-04. Upstream tag `v2.4.1` (2021-09-21). The MPRIS2
D-Bus interface is a stable freedesktop.org spec and the
playerctl CLI surface has not changed across the v2.x line;
re-verify only if a future major version (v3) reshapes the
template DSL.
