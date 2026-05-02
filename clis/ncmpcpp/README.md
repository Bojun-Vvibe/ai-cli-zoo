# ncmpcpp

> **The featureful ncurses TUI client for MPD (Music Player
> Daemon)** — playlist editor, browser, search, library
> view, lyrics fetcher, visualiser (`vis`), tag editor, and a
> media-library sort/filter pane, all driven from a vim-ish
> keymap (`j`/`k`/`/`/`Enter`/`a`/`s`) over MPD's plain-text
> TCP protocol on `localhost:6600`. Pinned to **0.10.1**
> ([COPYING](https://github.com/ncmpcpp/ncmpcpp/blob/0.10.1/COPYING),
> GPL-2.0-or-later).

Source: <https://github.com/ncmpcpp/ncmpcpp>

## TL;DR

`ncmpcpp` is a *client*, not a player: the actual decoding,
gapless mixing, ALSA/Pipewire output, ReplayGain, library
indexing, and HTTP streaming all live in `mpd` (a long-lived
local or remote daemon), and `ncmpcpp` is one of many
keyboard-driven front-ends speaking the MPD wire protocol.
That separation is the entire appeal — the daemon survives
your tmux session dying, your laptop sleeping, your `ssh`
disconnecting, and your terminal emulator crashing; the TUI
is a 5 MB binary you reattach whenever you want. Out of
the box you get: a Playlist screen with drag-reorder
(`m`/`M`), a Browse screen (filesystem view of your library
root), a Search Engine (`/`-prefixed live search across
artist/album/title/genre/comment), a Media Library three-pane
column view (artists → albums → tracks), a Tag Editor that
writes ID3v2 / Vorbis / FLAC tags in place, a Lyrics screen
that fetches from configured providers, a real-time spectrum
/ wave / ellipse Visualiser (`vis`, requires the daemon's
FIFO output enabled), and an Outputs screen that toggles
ALSA / PulseAudio / Pipewire sinks the daemon exposes. All
fully rebindable via `~/.config/ncmpcpp/bindings`; colours
in `~/.config/ncmpcpp/config`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install ncmpcpp mpd

# Debian / Ubuntu
sudo apt install ncmpcpp mpd

# Arch
sudo pacman -S ncmpcpp mpd

# Fedora
sudo dnf install ncmpcpp mpd

# Build from source (autotools, requires libmpdclient + ncurses + boost)
git clone --depth 1 --branch 0.10.1 \
  https://github.com/ncmpcpp/ncmpcpp.git
cd ncmpcpp && ./autogen.sh && \
  ./configure --enable-clock --enable-visualizer --with-taglib && \
  make && sudo make install
```

## Usage

```bash
# One-time MPD bring-up (per-user daemon, no root)
mkdir -p ~/.config/mpd ~/.local/share/mpd/playlists
cat > ~/.config/mpd/mpd.conf <<'EOF'
music_directory     "~/Music"
playlist_directory  "~/.local/share/mpd/playlists"
db_file             "~/.local/share/mpd/database"
state_file          "~/.local/share/mpd/state"
sticker_file        "~/.local/share/mpd/sticker.sql"
bind_to_address     "127.0.0.1"
port                "6600"
audio_output { type "pipewire"  name "pw" }
EOF
mpd                            # one-shot foreground; or wire systemd --user
mpc update                     # rescan the library

# Launch the client
ncmpcpp

# In-app: 1 playlist, 2 browse, 3 search, 4 library,
#         5 playlist editor, 6 tag editor, 8 visualizer,
#         /  filter,  Enter add+play,  s stop,  > next,  < prev,
#         q quit (daemon keeps playing)

# Drive the same daemon from scripts (mpc is the shell counterpart)
mpc add  "Tycho/Awake/01 - Awake.flac"
mpc play
mpc volume 60
mpc status

# Connect to a remote MPD on your home server
MPD_HOST=mediabox.lan MPD_PORT=6600 ncmpcpp
```

## Why it's interesting

The MPD ecosystem is the longest-lived demonstration of the
"daemon + thin clients" design that almost everything else
in audio gave up on. The daemon owns the decoder graph, the
output sink, the library index, the playlist queue, and the
crossfade state; clients (`ncmpcpp`, `mpc`, `cantata`,
`rmpc`, mobile apps like MALP, web UIs like ympd) attach
read/write over a 200-line plain-text TCP protocol and
disappear without disturbing playback. For a terminal-resident
workflow that means: the same audio session survives `ssh`
reconnects to your home server, can be controlled from your
phone *and* a tmux pane *and* a hotkey daemon *and* a shell
script simultaneously, and never loses queue state when the
UI crashes. `ncmpcpp` is the heaviest, most-complete client
in that family — pick over [`mpc`](https://www.musicpd.org/clients/mpc/)
when you want the interactive library / tag-editor / visualiser
surface, not just a one-shot CLI; pick over `cantata` when
you want a TUI not a Qt GUI; pick over `rmpc` (modern Rust
TUI, smaller surface) when you want feature breadth and 15
years of accumulated keybinds + community configs (rmpc for
modern Rust + image previews, ncmpcpp for the kitchen-sink
default). Pairs naturally with `mpd` (mandatory backend),
`mpc` (scripting from shell), `mpDris2` / `mpd-mpris` (MPRIS2
bridge so `playerctl` works), [`beets`](../beets/) (library
tagging / import pipeline that writes the same tree `mpd`
indexes), and a hotkey daemon (`sxhkd` on X11, `swhkd` /
compositor binds on Wayland) for global media keys. Caveats
— hard prereq is a running MPD daemon (`brew install mpd`
isn't enough; you must write `mpd.conf` and point
`music_directory` at your library), the visualiser needs
MPD's `audio_output { type "fifo" … }` configured and a
matching `visualizer_data_source` in `ncmpcpp`'s config
(otherwise the screen stays blank), tag-editor writes are
in-place and immediate (back up before bulk ops, or pair
with [`beets`](../beets/) which tracks history), GPL-2.0-or-later
on the client side and GPL-2.0 on `mpd` itself, and there
are *no* GitHub Releases — installs come from distro packages
or `git checkout 0.10.1`.
