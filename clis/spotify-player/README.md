# spotify-player

> **A terminal Spotify client with full playback, search, and
> library control via the official Spotify Web API + librespot
> playback backend** — a Rust TUI that authenticates against
> your Spotify account, renders playlists / albums / artists /
> liked-songs / browse / lyrics in a multi-pane interface, and
> can either remote-control an existing Spotify Connect device
> or play audio itself via a built-in librespot streaming
> client. Pinned to **v0.23.0** ([LICENSE](https://github.com/aome510/spotify-player/blob/master/LICENSE),
> MIT).

Source: <https://github.com/aome510/spotify-player>

## TL;DR

`spotify-player` is what you reach for when you want Spotify but
do not want the Electron desktop app open all day eating RAM and
notifying you about podcasts. After a one-time OAuth handshake
through your browser (the binary opens a localhost callback,
Spotify redirects, token is cached under
`~/.cache/spotify-player/`), every subsequent launch drops you
into a multi-pane TUI: left side is a context list (playlists,
albums, recently played, browse categories), right side is the
track table for the current context, bottom is a now-playing
bar with track / artist / album / progress / volume / shuffle /
repeat. Keys are vim-flavored (`j`/`k` to move, `gg`/`G` to
jump, `/` to fuzzy-search the current list, `Enter` to play,
`d` to switch playback device, `s` to toggle shuffle).
`v0.23.0` adds an optional real-time audio visualisation pane.
The whole thing is one ~12 MB binary; the streaming backend is
optional (compile-time `streaming` feature, runtime requires
ALSA / PulseAudio / CoreAudio and a Spotify Premium account).

## Install

```bash
# Homebrew (macOS / Linux)
brew install spotify-player

# Cargo (default features = streaming + media-control + image)
cargo install spotify_player --locked

# Cargo, remote-control only (no librespot, smaller binary,
# no audio backend dependency — ideal for a remote / headless
# box that just controls another device)
cargo install spotify_player --locked --no-default-features \
    --features pulseaudio-backend  # or rodio-backend / portaudio-backend / ...

# Arch Linux
pacman -S spotify-player           # community repo
# or AUR: yay -S spotify-player-git

# Nix
nix-env -iA nixpkgs.spotify-player

# Pre-built release tarball
curl -LO https://github.com/aome510/spotify-player/releases/download/v0.23.0/spotify_player-aarch64-apple-darwin.tar.gz
tar xf spotify_player-aarch64-apple-darwin.tar.gz
sudo install spotify_player /usr/local/bin/

# verify
spotify_player --version    # spotify_player 0.23.0
```

First launch opens a browser OAuth flow; you will be asked for
a **Spotify Developer client_id** (free to register at
<https://developer.spotify.com/dashboard>; the binary tells you
exactly which redirect URI to whitelist —
`http://127.0.0.1:8989/callback`). Token + refresh-token live
in `~/.cache/spotify-player/`; config (theme, keybinds, default
device, layout) in `~/.config/spotify-player/app.toml`.

## Use it for

```bash
# Just launch the TUI
spotify_player

# Headless remote-control mode: send a command to a running
# instance via the local CLI socket
spotify_player playback play-pause
spotify_player playback next
spotify_player playback volume +10
spotify_player playback shuffle
spotify_player playback repeat track
spotify_player playback seek +30s

# Pick what to play from the CLI (no TUI needed)
spotify_player playback start context --type=playlist \
    --id=37i9dQZF1DXcBWIGoYBM5M       # Today's Top Hits
spotify_player playback start track \
    --uri=spotify:track:4cOdK2wGLETKBW3PvgPWqT
spotify_player search "miles davis kind of blue"

# Switch which Spotify Connect device is the active sink
# (your phone, a Sonos, a Chromecast, this laptop's librespot
# instance)
spotify_player playback connect
# → opens an interactive picker

# Get current playback as JSON (good for status bars)
spotify_player get key playback
spotify_player get key playback | jq '.item.name + " — " + .item.artists[0].name'
```

`spotify-player` exposes a small **CLI command surface** in
addition to the TUI, which is what makes it scriptable: the
running instance opens a local Unix socket
(`~/.cache/spotify-player/app.sock`), and `spotify_player <cmd>`
in another shell sends a command to it and exits. That is the
hook for tying it into a tmux status line, a polybar / waybar
module, an `sxhkd` media-key binding, or an LLM agent that
wants to "pause music while I'm in a meeting".

## Why include it in a CLI catalog

1. **It's a CLI surface for a service that is normally
   GUI-only.** Spotify ships a desktop app (Electron, ~300 MB
   RAM idle), a web player, and mobile apps; there is no
   first-party CLI. `spotify-player` fills that gap with a
   single binary that exercises the same Web API, which means
   a terminal-first user (or an agent) gets *all* of Spotify's
   library / search / browse / queue management without leaving
   the shell, and a Premium subscriber gets actual audio output
   too.
2. **The CLI sub-commands are the integration story.** Most
   TUIs are end-points (you stare at them); `spotify-player`
   doubles as a daemon you can poke from outside. Chaining
   `spotify_player get key playback | jq ...` into a tmux
   status line or a `dunstify` notifier is the same shape as
   chaining `git status -s` into a prompt — that's rare for a
   media app and is what makes it useful in an agent / dotfiles
   context.
3. **Two playback modes from one binary.** Compile with the
   `streaming` feature and it embeds librespot, becoming its
   own Spotify Connect speaker (audio comes out of the machine
   running the binary). Compile without, and it's a pure
   remote-control for whichever Connect device is already
   playing (your phone, a smart speaker, the desktop app on
   another machine). Same UI, same keybinds, same CLI — you
   pick the role at install time.

For an LLM-CLI workflow, the JSON-emitting sub-commands
(`spotify_player get key playback`, `spotify_player get key
queue`) give an agent a structured handle on "what's playing"
without scraping a TUI, and the `playback` verbs let it `pause`
during a long-running task or `volume -50` before reading a
notification aloud.

## Vs Already Cataloged

- **Vs [`ncmpcpp`](../ncmpcpp/):** orthogonal — `ncmpcpp` is a
  TUI client for **MPD** (Music Player Daemon, plays files from
  your local library / a SMB share / an MPD server you run);
  `spotify-player` is a TUI client for **Spotify** (streams from
  a commercial service, requires an account, plays the catalog
  Spotify licences). If your music is files you own, you want
  `ncmpcpp` + `mpd`. If your music is "whatever is on Spotify
  this month", you want `spotify-player`. They share zero
  backend code and zero auth surface.
- **Vs [`mpv`](../mpv/):** orthogonal — `mpv` plays a URL or
  file you already have (with `yt-dlp` it can play a YouTube
  link); `spotify-player` browses, searches, and queues a
  *catalog* (library, playlists, browse pages, recommendations,
  recently-played) and then plays from it. `mpv` has no concept
  of "my liked songs". `spotify-player` has no concept of "play
  this arbitrary `.mkv`".
- **Vs [`spotify-tui`](../spotify-tui/) (already cataloged):**
  closest peer — same idea, older project (`Rigellute/spotify-tui`,
  archived 2022). `spotify-player` is the actively-maintained
  successor: adds librespot streaming (spotify-tui was
  remote-control only), adds a CLI command socket, adds lyrics
  and browse pages, and is still receiving releases (v0.23.0
  shipped 2026-03). New users should pick `spotify-player`;
  existing `spotify-tui` users get a near-identical keybind set.
- **Vs the official Spotify desktop app (not cataloged, not a
  CLI):** desktop app has the full visual catalog and podcasts;
  `spotify-player` is keyboard-only and skips podcasts (not
  exposed by the Web API the same way). Trade RAM and a Connect
  device count for ~12 MB binary and `tmux`-friendly layout.

## Caveats

- **Requires a Spotify Developer app for the OAuth client_id.**
  Free, takes ~3 minutes (dashboard → Create app → copy
  client_id, paste into the prompt on first launch, whitelist
  the `127.0.0.1:8989/callback` redirect). One-time, but it is
  not zero-friction — there is no built-in "shared client_id"
  to avoid making users provision their own (Spotify's TOS
  effectively forbids it).
- **Audio playback (the `streaming` feature) requires Spotify
  Premium.** Free-tier accounts can use `spotify-player` as a
  remote control / browser only; trying to start playback on
  the local librespot device returns a Spotify Connect error.
- **librespot's audio backend is platform-specific.** On Linux
  you pick `pulseaudio-backend`, `alsa-backend`, or
  `rodio-backend` at compile time; on macOS the default
  `rodio-backend` works; on Windows likewise. Mismatching the
  feature to the system gives you a binary that runs and shows
  the TUI but is silent — check the install command above.
- **Web API rate limits leak through.** Heavy use (rapid
  search, scrolling browse pages) can hit Spotify's per-app
  rate limits and the TUI shows transient `429` errors in the
  status line; usually self-resolves within seconds, but during
  a hammering session the now-playing bar can lag.
- **Lyrics are best-effort.** Lyrics come from the Spotify
  client API, which is intermittently restricted by region /
  account type; missing lyrics show as an empty pane, not an
  error. Do not pick `spotify-player` *because* of lyrics.
- **No podcast / video-podcast UI.** The Web API surfaces
  episodes, but `spotify-player`'s pane layout is built around
  tracks; podcast support is partial. If you live in podcasts,
  the desktop app is still the path of least resistance.
