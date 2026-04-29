# termusic

> Snapshot date: 2026-04. Upstream: <https://github.com/tramhao/termusic>

A terminal music player in Rust, modelled on `ncmpcpp` / `cmus` but
shipped as a single static binary with no MPD dependency. Plays
local FLAC/MP3/OGG/AAC/WAV/M4A libraries, internet radio streams,
and (optional, behind feature flags) podcast feeds; renders album
art with kitty / sixel / iTerm graphics, scrobbles to Last.fm, and
fetches synced lyrics. Acts as the catalog's reference "ambient
audio without leaving the terminal" entry — same slot `cmus` filled
historically but with a modern Rust toolchain and TUI library.

## 1. Install

- `brew install termusic` — Homebrew (macOS / Linux); ships both
  `termusic` (the TUI client) and `termusic-server` (playback daemon)
- `cargo install termusic --locked` — from crates.io (requires a
  C toolchain plus `pkg-config`, `libasound2-dev` on Linux,
  `gst-plugins-*` if you opt into the GStreamer backend)
- `pacman -S termusic`, `nix-env -iA nixpkgs.termusic`
- Pre-built `.tar.gz` per platform on the
  [releases page](https://github.com/tramhao/termusic/releases)
- Backends: defaults to `rusty` (Symphonia-based, pure Rust); opt
  into `mpv` or `gstreamer` via cargo features for broader codec
  coverage.

## 2. Version pin

**v0.12.1**. Verify with `termusic --version`.

## 3. License

Dual-licensed MIT AND GPL-3.0-or-later. SPDX:
`MIT AND GPL-3.0-or-later`. Full texts at
[`LICENSE`](https://github.com/tramhao/termusic/blob/master/LICENSE)
and the per-component headers in the repository tree.

## 4. What it does

- Two-pane TUI: library tree on the left (filesystem-rooted at the
  configured `music_dir`), playlist / queue on the right; `tab`
  swaps focus, vim keys navigate.
- Playback verbs: `space` play/pause, `n/p` next/prev, `s` stop,
  `+/-` volume, `</>`  seek, `r` repeat mode, `z` shuffle, `m`
  mute. Gapless playback via the `rusty` backend.
- Library actions: `a` add to queue, `A` add album, `l` load
  playlist (M3U/M3U8/PLS), `S` save current queue, `/` fuzzy filter,
  `D` delete file from disk (with confirm).
- Tag editor (`t`): edit ID3v2 / Vorbis comments / MP4 atoms in
  place; bulk-rename via the configurable `file_format` template.
- Album art rendered via the terminal graphics protocol your
  terminal supports (kitty, iTerm2, sixel-capable wezterm/foot);
  falls back to ASCII when none is detected.
- Synced lyrics view (`L`) — pulls from configured providers,
  scrolls with the playhead.
- Internet radio + podcast (behind feature flags); subscriptions
  stored in `~/.config/termusic/`.
- Last.fm scrobbling and a built-in MPRIS bridge on Linux so global
  media keys work.

## 5. Install & first use

```bash
brew install termusic
mkdir -p ~/Music
# point termusic at your library on first run via $EDITOR on the
# generated ~/.config/termusic/config.toml (set `music_dir`)
termusic                              # starts the daemon + TUI
# inside the TUI:
#   tab           toggle library / playlist focus
#   enter         add highlighted track / album to the queue
#   space         play / pause
#   /Radiohead    fuzzy filter the library
#   t             edit tags on the highlighted track
#   q             quit
```

## 6. Example: headless daemon + remote control

```bash
# 1. start the playback daemon detached (handy on a media-server box)
termusic-server --daemon

# 2. drive it from another terminal / SSH session via the TUI:
termusic                              # connects to the running daemon

# 3. or script it via the gRPC interface the daemon exposes
#    (see `termusic-server --help` for the socket path); the
#    `termusic` client itself is the reference consumer.
```

## 7. Why it matters in this catalog

Most CLIs in this catalog are about producing artifacts — code,
diffs, tickets, dashboards. `termusic` is here as the
counterweight: an explicit "long-running terminal-resident program
that is *not* an editor, build tool, or AI agent" entry, useful
when you want to keep a dev workstation fully keyboard-driven and
not context-switch to a graphical music app. It also serves as a
clean reference implementation of the modern Rust TUI stack
(ratatui + tokio + symphonia) for catalog readers comparing TUI
architectures across `lazygit`, `gitui`, [`taskwarrior-tui`](../taskwarrior-tui/),
and [`gpg-tui`](../gpg-tui/).

## 8. Alternatives

- `cmus` — C, decade-stable, MPD-style; the historical reference.
  Pick `cmus` if you need maximum codec coverage via FFmpeg and
  don't want the dual-binary client/server split.
- `ncmpcpp` — needs MPD running; richer ecosystem (visualiser,
  lyrics plugins) but heavier setup.
- `mpd` + `mpc` — server + thin client; pair with `ncmpcpp` for
  the classic stack.
- `spotify-tui` — Spotify-only; web API auth, no local files.
- `mpv` with a TUI keymap — minimalist alternative for when you
  just want a queue-of-files player and not a library browser.

## 9. Telemetry

No first-party analytics. Outbound network only when you explicitly
opt in: Last.fm scrobbling (only if the API key is configured),
internet-radio stream URLs you add yourself, podcast feed refreshes
for subscriptions you add yourself, and lyrics-provider lookups
when the lyrics view is opened. Air-gapped use is the default.
