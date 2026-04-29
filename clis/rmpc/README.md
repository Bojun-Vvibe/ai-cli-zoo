# rmpc

> **A modern, configurable terminal client for the Music
> Player Daemon (MPD)** — replaces aging clients like `ncmpcpp`
> and `cmus`-as-MPD-frontend with a Rust TUI that supports
> inline album art via Kitty / iTerm2 / Sixel image protocols,
> Lua-scripted keybindings and themes, lyrics panels, last.fm
> scrobbling, and on-the-fly profile switching across multiple
> MPD instances. Pinned to **v0.11.0**
> ([LICENSE](https://github.com/mierak/rmpc/blob/master/LICENSE),
> BSD-3-Clause).

Source: <https://github.com/mierak/rmpc>

## TL;DR

`rmpc` is what you reach for when you have a self-hosted MPD
server (a Raspberry Pi in the closet, a Navidrome-adjacent
setup, a desktop running `mpd.service` against a 60k-track
library) and you want a *good* terminal client — meaning one
that renders album art inline in the terminal instead of as
a popup, that you can theme without learning a bespoke INI
dialect, and that is being actively maintained in 2026. The
incumbent in this space, `ncmpcpp`, is a venerable C++
client that has not had a substantive UX update in years and
cannot render images at all. `rmpc` is a single Rust binary,
configured in [RON](https://github.com/ron-rs/ron) (Rust
Object Notation) with hot-reload on `SIGUSR1`, and ships
with first-class image support via the Kitty graphics
protocol, iTerm2 inline images, and Sixel — so the album
cover for the currently-playing track shows up *in your
terminal pane* at a real resolution, not as ASCII art. It
also supports Lua scripting for custom keybindings and
status-bar segments, multi-profile config (different MPD
hosts on different keys), search across the entire library,
queue editing with undo, lyrics fetched from external
sources, and last.fm scrobbling. The design goal is "the
client that finally retires `ncmpcpp` for the next decade",
and on a kitty / wezterm / ghostty terminal it succeeds.

## When to choose

- **You already run MPD** for your music library — at home,
  on a NAS, on a Pi — and your current client is
  `ncmpcpp`, `cmus` (which doesn't speak MPD natively),
  `mpc` (CLI-only), or a web frontend you don't love.
- **Your terminal supports Kitty graphics, iTerm2 images,
  or Sixel** (Kitty, WezTerm, Ghostty, Konsole, foot, iTerm2,
  Black Box, recent xterm). Album art rendering is the
  flagship feature; without image support you lose the main
  reason to switch.
- **You want config-as-code** — RON files for layout,
  keybindings, theme, and Lua snippets for dynamic status —
  rather than a flat INI with magic comment syntax.
- **You manage multiple MPD instances** — e.g., `home` and
  `office` and `livingroom` — and want one binary that
  switches between them with a hotkey instead of editing
  config to reconnect.

## When NOT to choose

- **You don't run MPD** — `rmpc` is *only* an MPD client. It
  does not stream from Spotify, Apple Music, YouTube Music,
  Tidal, Plex, Jellyfin, Subsonic, or local files directly.
  For Spotify use [`spotify-tui`](https://github.com/Rigellute/spotify-tui)
  (effectively unmaintained) or `spotify_player`. For
  Subsonic / Navidrome use [`termusic`](../termusic/),
  `stmp`, or `super-sonic`. For YouTube Music use `ytmdl`.
- **Your terminal cannot render images** (basic Linux
  console, tmux without the image-passthrough patch, older
  gnome-terminal, Apple Terminal.app) — you can still run
  `rmpc` but you lose the cover-art differentiator and
  `ncmpcpp` becomes a reasonable alternative again.
- **You want a GUI** — use [Cantata](https://github.com/CDrummond/cantata)
  or the official MPD web frontends.
- **You need offline-first track management** with tag
  editing — `rmpc` is a *player*, not a library manager.
  Use [Beets](https://beets.io) for tagging / organizing
  the underlying files; let `rmpc` consume the result.

## Install

```sh
# Arch Linux (AUR)
yay -S rmpc

# Nix
nix-shell -p rmpc

# Cargo (any platform with Rust 1.78+)
cargo install --locked --tag v0.11.0 \
  --git https://github.com/mierak/rmpc

# from a release binary (Linux x86_64)
curl -LO https://github.com/mierak/rmpc/releases/download/v0.11.0/rmpc-x86_64-unknown-linux-gnu.tar.gz
tar xzf rmpc-x86_64-unknown-linux-gnu.tar.gz
sudo install rmpc /usr/local/bin/

# generate a starter config (writes to ~/.config/rmpc/config.ron)
rmpc config > ~/.config/rmpc/config.ron

# verify
rmpc --version    # rmpc 0.11.0
```

## What it does

- Full MPD client: queue management, library browsing,
  playlist editing, search, seeking, crossfade, replaygain,
  output device switching.
- Inline album-art rendering via Kitty graphics protocol,
  iTerm2 images, Sixel — with automatic detection and
  fallback.
- RON-based config with live reload via `SIGUSR1`.
- Lua scripting for custom keybindings, status-bar
  components, and event hooks (`on_song_change`).
- Lyrics panel (LRC fetching + scrolling sync to playback
  position).
- last.fm scrobbling, MPRIS bridging on Linux for
  media-key control from the desktop environment.
- Multi-profile: define multiple MPD hosts/passwords and
  switch with a keybind.
- Themable: ship-with palettes plus a documented theme
  format.

## pew-related use cases

- **Headphones-on coding sessions**: keep a tmux pane with
  `rmpc` next to your editor; album art makes "what's
  playing" answerable at a glance without alt-tabbing to
  Spotify.
- **Self-hosted music stack on a homelab Pi**: pair MPD on
  a Raspberry Pi (`mpd.service`) with `rmpc` on every
  laptop / desktop pointing at it via SSH or directly to
  `:6600`; one library, many controllers.
- **Replace `ncmpcpp` everywhere**: same protocol, modern
  TUI, image support — drop-in for users who have been
  running ncmpcpp for a decade and are ready to move on.
- **Demo / screenshot recordings** for blog posts about
  CLI-first workflows: pair with [`asciinema`](../asciinema/)
  or [`freeze`](../freeze/); the album-art cell makes the
  recording visually distinct from generic terminal demos.

## Comparable tools

- **`ncmpcpp`** ([upstream](https://github.com/ncmpcpp/ncmpcpp))
  — the long-standing C++ MPD client. Stable, ubiquitous,
  packaged everywhere, but no inline images and the UX is
  showing its age. `rmpc` is the modern replacement.
- **`mpc`** — official MPD CLI, no TUI, fine for scripting
  (`mpc play`, `mpc next`) but not a daily-driver client.
- **[`termusic`](../termusic/)** — terminal player but with
  its *own* backend (does not speak MPD). Choose `termusic`
  if you don't already run MPD; choose `rmpc` if you do.
- **`cmus`** — single-process player, not an MPD client at
  all; if you want a self-contained TUI player without a
  daemon, use `cmus`. If you want a *client* for an existing
  MPD daemon, use `rmpc`.
- **Cantata** — Qt GUI client for MPD; good for
  desktop-first users, irrelevant if you live in a terminal.
