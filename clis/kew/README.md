# kew

> **Terminal music player for local libraries** — a single C
> binary that indexes a directory of audio files (FLAC / MP3 /
> Opus / Ogg Vorbis / WAV / AAC / m4a) once into a fast in-memory
> tree, then exposes a curses-style TUI where you fuzzy-search
> the library by track / album / artist (`/` then type), queue
> the matched playlist, and play it through PulseAudio /
> PipeWire / ALSA / CoreAudio with gapless playback, optional
> visualisation, MPRIS D-Bus controls, and Last.fm scrobbling —
> all without a daemon, a database server, or a web UI. Pinned
> to **v4.0.0** (commit
> `649677c9fc64f825d1e458b581f5ef4bfd416d68`,
> [LICENSE](https://github.com/ravachol/kew/blob/main/LICENSE),
> GPL-2.0-or-later).

Source: <https://github.com/ravachol/kew>

## TL;DR

`kew` is the right answer when you have a local audio library
(`~/Music/` with thousands of FLAC / Opus rips), live in a
terminal, and find Spotify / cmus / mpd-and-ncmpcpp all wrong
for different reasons: Spotify has no offline lossless and no
local-file priority; `cmus` is excellent but its playlist model
is "you maintain the queue manually"; `mpd` + `ncmpcpp` is two
processes and a config file before you can press play. `kew` is
one binary, one command (`kew <fuzzy query>`), and music starts.

`kew artist=radiohead album=in-rainbows` queues that album. `kew
weather` searches every tag field across the indexed library and
queues whatever matches "weather" (a track, an album, a playlist,
an artist). Inside the TUI, `space` is play/pause, `left`/`right`
seek, `up`/`down` adjust volume, `n`/`p` next/previous track,
`/` opens a fuzzy search across the library, `Tab` cycles
playlist / library / settings / track-info views, `q` quits.

The library scan is a one-time `kew --refresh-library` walk that
caches metadata in `~/.config/kew/cache.db` (small SQLite); after
that, startup is sub-100ms even on a 50 GB library. Audio
backend auto-detects PulseAudio / PipeWire / ALSA on Linux and
CoreAudio on macOS; gapless playback is on by default for
formats that support it (FLAC / Opus / Vorbis).

## Install

```bash
# Homebrew (macOS / Linux)
brew install kew

# Arch Linux (AUR)
yay -S kew
# or: paru -S kew

# Fedora / openSUSE / Ubuntu — build from source
git clone --branch v4.0.0 https://github.com/ravachol/kew
cd kew
make
sudo make install

# verify
kew --version    # 4.0.0
```

Build deps on Linux: `pkg-config`, `libavformat-dev`,
`libavcodec-dev`, `libavutil-dev`, `libswresample-dev`,
`libopus-dev`, `libvorbis-dev`, `libogg-dev`, `libfaad-dev`,
`libtag1-dev`, one of `libpulse-dev` / `libpipewire-0.3-dev` /
`libasound2-dev`.

## License

GPL-2.0-or-later — see
[LICENSE](https://github.com/ravachol/kew/blob/main/LICENSE).
Source modifications must be redistributed under the same
licence; binary use against your own library is unrestricted.

## Hot keybinds

- `space` — play/pause
- `left` / `right` — seek backward / forward 5s
- `up` / `down` — volume up / down
- `n` / `p` — next / previous track in the queue
- `/` — fuzzy search across the indexed library
- `Tab` — cycle views (playlist / library / settings /
  track-info)
- `s` — toggle shuffle
- `r` — toggle repeat (off / one / all)
- `+` / `-` — change playback speed (0.5× to 2×)
- `q` — quit

## Why use it

- **Zero-daemon, zero-config first run.** `mpd` + `ncmpcpp` is
  the canonical "TUI music player on Linux" recipe, but it is
  two processes, a `mpd.conf`, a `ncmpcpp` config, and a music
  directory declaration before you can press play. `kew
  ~/Music` indexes and plays in one command.
- **Fuzzy library search is the input model**, not the
  afterthought. `cmus` makes you maintain the queue manually;
  `kew` makes you describe what you want in tag-shaped query
  syntax (`artist=`, `album=`, `genre=`) or free text and
  builds the queue from the match.
- **Gapless playback for FLAC / Opus / Vorbis** is default-on,
  not a config switch. The right behaviour for a classical or
  electronic album that crossfades on the source.
- **MPRIS D-Bus controls** mean media-key keyboards, GNOME /
  KDE / Hyprland status bars, and `playerctl` scripts work
  against `kew` the same way they work against Spotify or
  Rhythmbox — pause from a keyboard `XF86AudioPause` press,
  display the now-playing track in a bar widget, etc.

## Vs Already Cataloged

- **Vs [`spotify-tui`](../spotify-tui/) /
  [`spotify-player`](../spotify-player/):** orthogonal — those
  are clients for the Spotify streaming service (need a Spotify
  Premium account, no local-file priority). `kew` is for a local
  library you own. Run both: `spotify-player` for streaming
  discovery, `kew` for the FLAC rips of the albums you bought.
- **Vs [`ncmpcpp`](../ncmpcpp/):** ncmpcpp is the established
  ncurses front-end for `mpd`, requires running `mpd` as a
  daemon, configures via two files. `kew` is one binary, no
  daemon, fuzzy-search-first input model. Pick ncmpcpp for the
  mature plugin ecosystem and mpd's network-server architecture
  (multiple clients, remote control); pick kew for "I want
  music in 30 seconds on a fresh box."
- **Vs [`cava`](../cava/):** orthogonal — `cava` is a standalone
  audio visualiser that reads from PulseAudio / PipeWire and
  renders bars. `kew` has its own optional visualiser; run
  `cava` next to it if you want a richer animation.
- **Vs [`beets`](../beets/):** complementary — `beets` is the
  library *organiser* (auto-tag from MusicBrainz, normalise
  filenames, fetch album art); `kew` is the *player*. Run beets
  to canonicalise the library, then `kew` to play it.

## Caveats

- **GPL-2.0-or-later**, not MIT/Apache. Binary redistribution is
  fine for any use (personal, commercial, embedded). Forks of
  the source or static linking against modified kew code carry
  GPL obligations.
- **Local files only.** No streaming protocols, no Spotify /
  Apple Music / Tidal integration, no internet radio. `kew` is
  deliberately scoped to "play the audio files on this disk."
- **Library scan must be re-run after adding files**
  (`kew --refresh-library`). The cache is not a file-watcher
  daemon (intentional — keeps the "no background process" invariant).
- **macOS audio backend is CoreAudio only**; PipeWire / PulseAudio
  paths are Linux-only. No Windows support today.
- **Visualiser modes are basic** compared to dedicated tools
  (`cava`, `ncmpcpp`'s spectrum). Use kew's visualiser as a
  glance-check that audio is flowing; pipe to `cava` if you
  want a richer rendering.
- **Calendar-style versioning is not used**; semver `v4.0.0`
  signals a breaking config / cache layout change from `v3.x`.
  After upgrading, re-run `kew --refresh-library` if the queue
  starts empty.
