# spotify-tui

> **A terminal UI for Spotify** — a single Rust
> binary that talks to the Spotify Web API and
> presents your library, playlists, devices, and
> currently-playing track as a Bubble-Tea-style
> keyboard-driven TUI, so you can pause, skip,
> queue, and search without leaving the terminal.
> Pinned to **v0.25.0**
> ([LICENSE](https://github.com/Rigellute/spotify-tui/blob/master/LICENSE),
> MIT, © Alexander Keliris).

Source: <https://github.com/Rigellute/spotify-tui>

## TL;DR

`spt` (the binary name) is what you reach for when
you want full Spotify Premium control without
giving the desktop client a slot in your dock or
your tmux layout. It uses the Spotify Web API
under the hood, which means **it does not stream
audio itself** — you still need a playback
endpoint somewhere on the network (the official
desktop client, the iOS/Android app, a Chromecast,
a Sonos, or [`librespot`](https://github.com/librespot-org/librespot)
running headless on the same box) — but everything
*about* playback (queue, library, search, device
switching, playlist editing, follow/unfollow,
recently-played, top tracks, artist pages, the
"radio" generated playlists) is driven from the
keyboard via vim-style bindings: `h/j/k/l` to
navigate panes, `/` to search, `space` to
play/pause, `n/p` to skip, `d` to switch device,
`s` to shuffle, `r` to repeat, `?` for the help
overlay. First launch walks you through the
OAuth dance — you create a Spotify developer app,
paste in the client ID and secret, and `spt`
opens a browser to capture the redirect; the
refresh token is then cached under
`~/.config/spotify-tui/` and reused forever. Once
authed, the TUI gives you the same surface area
as the desktop client minus the album-art carousel
and the social/feed cruft.

## Install

```bash
# Homebrew (macOS / Linux)
brew install spotify-tui

# Cargo (any platform with a Rust toolchain)
cargo install spotify-tui

# Snap (Linux)
snap install spt

# Arch Linux (community)
pacman -S spotify-tui

# Nix
nix-env -iA nixpkgs.spotify-tui

# Scoop (Windows)
scoop install spotify-tui

# pre-built binaries on the releases page
# https://github.com/Rigellute/spotify-tui/releases

# verify
spt --version    # spt 0.25.0
```

## Basic usage

```bash
# launch the TUI (first run prompts for client ID/secret)
spt

# play a track / playlist / album by URI from the shell
spt play --uri spotify:track:6rqhFgbbKwnb9MLmUQDhG6

# play context (artist top tracks, album, playlist)
spt play --uri spotify:album:6JWc4iAiJ9FjyK0B59ABb4

# enqueue a track
spt play --uri spotify:track:6rqhFgbbKwnb9MLmUQDhG6 --queue

# list available playback devices
spt list -d

# transfer playback to a specific device
spt playback --transfer "Living Room Speaker"

# print currently playing track in shell-friendly form
spt playback --status --format "%a - %t"

# pause / play / next / previous from the CLI
spt playback --toggle
spt playback --next
spt playback --previous

# search interactively
#   from inside the TUI: press / then type
```

A small but useful subset of the TUI is exposed
as subcommands (`spt play`, `spt playback`,
`spt list`) so you can wire it into shell scripts,
keybindings, or status-bar polls without
launching the full TUI.

## When to choose

- **Your workflow lives in tmux/zellij and the
  Spotify desktop app feels heavy** — `spt` runs
  in a pane, sips RAM, and gives you the whole
  library in keyboard reach.
- **You stream from a headless Linux box via
  `librespot` and want a real client to drive it**
   — `spt` + `librespot` is the canonical
  "Spotify on a Raspberry Pi without a screen"
  combo: `librespot` advertises the device on the
  network, `spt` selects it with one keystroke and
  controls playback from your laptop.
- **You want a status-bar "now playing" without
  scraping `dbus` or AppleScript** —
  `spt playback --status --format "%t - %a"` is a
  one-line shell call you can poll from i3blocks,
  polybar, tmux status-right, sketchybar, or your
  zsh prompt.
- **You script playlist curation** —
  `spt play --uri ...` and the TUI's `e`-to-edit
  on a playlist make it usable as a daily-driver
  playlist builder, especially with shell history
  for URIs.

## When NOT to choose

- **You don't have Spotify Premium** — the Web API
  endpoints `spt` uses for playback control
  (transfer, play URI, queue, seek, shuffle,
  repeat) are Premium-only. Free accounts can read
  library data but the TUI's playback pane will
  refuse to act.
- **You expect `spt` to play audio itself** — it
  doesn't. It's a remote control. You need the
  desktop client, the mobile app, a Connect-capable
  speaker, or a `librespot` daemon somewhere
  reachable.
- **You want lyrics / album art / podcast
  chapters in-pane** — the TUI is text-only and
  intentionally so. For lyrics overlay try
  [`sptlrx`](https://github.com/raitonoberu/sptlrx).
- **You don't want to register a Spotify developer
  app** — the OAuth flow requires you to create
  your own app at developer.spotify.com to obtain
  a client ID/secret. There's no shared/embedded
  credential. If that's a dealbreaker, the
  desktop client is the only escape hatch.

## Why it fits the zoo

The zoo's "TUI front-end for a hosted SaaS"
cluster ([`gh-dash`](../gh-dash/),
[`atac`](../atac/),
[`posting`](../posting/),
[`slumber`](../slumber/),
[`bruno`](../bruno/) for HTTP,
[`atuin`](../atuin/) for sync) is
about *not having to switch context out of the
terminal to drive a service that has a perfectly
good web UI you just don't want to look at*.
`spotify-tui` is the music-streaming entry: same
recipe (OAuth-once, keyboard-driven, `--format`
flag for shell integration), same payoff (one
less browser tab, one less Electron window). It
also slots next to the zoo's offline music TUIs
([`termusic`](../termusic/),
[`rmpc`](../rmpc/)) as the "I have a Spotify
subscription and I'm not migrating my library"
counterpoint to the local-first crowd.

## Upstream pointers

- Repo: <https://github.com/Rigellute/spotify-tui>
- Release notes: <https://github.com/Rigellute/spotify-tui/releases>
- License: [MIT](https://github.com/Rigellute/spotify-tui/blob/master/LICENSE)
- Changelog: <https://github.com/Rigellute/spotify-tui/blob/master/CHANGELOG.md>
- Author: [@Rigellute](https://github.com/Rigellute)
- Companion project: [`librespot`](https://github.com/librespot-org/librespot)
  — open-source Spotify Connect client library, the
  usual headless playback endpoint paired with `spt`.
