# mpv

> **A scriptable, terminal-controllable media player built on
> the MPlayer / mplayer2 lineage** — one binary that plays
> nearly every container and codec FFmpeg knows
> (`mp4`/`mkv`/`webm`/`flac`/`opus`/HLS/DASH/RTSP/network
> URLs), exposes a JSON-over-IPC socket so any shell or editor
> can drive it (`{"command":["seek",30]}`), and runs headless
> in a tmux pane via `--vo=tct` (true-colour terminal output)
> or `--vo=caca` for ASCII playback over SSH. Pinned to
> **v0.41.0**
> ([Copyright](https://github.com/mpv-player/mpv/blob/v0.41.0/Copyright),
> GPL-2.0-or-later by default; LGPL-2.1-or-later with
> `-Dgpl=false`).

Source: <https://github.com/mpv-player/mpv>

## TL;DR

`mpv` is the player you reach for when "play this file" is
just step one and the rest of the requirement is *control it
from somewhere else*. The IPC socket
(`--input-ipc-server=/tmp/mpv.sock`) accepts JSON commands
and emits property-change events, so a 30-line shell script
can build a "now-playing" status bar, a global hotkey daemon,
or a `yt-dlp`-fed radio queue without touching the GUI.
Lua / JavaScript scripts under `~/.config/mpv/scripts/` get
the same surface in-process — popular ones include
`mpv-mpris` (D-Bus / MPRIS2 so playerctl works),
`autoload.lua` (auto-queue the rest of a directory),
`sponsorblock.lua` (skip YouTube sponsor segments), and
`thumbfast.lua` (seekbar previews). On the rendering side
the OpenGL / Vulkan video output (`--vo=gpu-next`) does
high-quality scaling (`--scale=ewa_lanczossharp`), HDR
tone-mapping, colour-managed output and 10-bit pipelines that
matter for film work; for a terminal-only host `--vo=tct`
emits Unicode half-blocks at 24-bit colour and `--vo=caca`
falls back to ASCII art over SSH. Reads almost everything
FFmpeg can demux, including network sources (`mpv
https://example.com/stream.m3u8`), and pairs naturally with
`yt-dlp` (`mpv 'ytdl://...'` invokes it transparently).

## Install

```bash
# Homebrew (macOS / Linux)
brew install mpv

# Debian / Ubuntu
sudo apt install mpv

# Arch
sudo pacman -S mpv

# Build from source (Meson, FFmpeg + libplacebo required)
git clone --depth 1 --branch v0.41.0 \
  https://github.com/mpv-player/mpv.git
cd mpv && meson setup build && meson compile -C build
```

## Usage

```bash
# Play a local file or a URL
mpv video.mkv
mpv https://example.com/stream.m3u8

# Headless playback in a terminal (true-colour blocks)
mpv --vo=tct --really-quiet song.mp4

# Audio-only player driven by an external IPC socket
mpv --no-video --idle=yes \
    --input-ipc-server=/tmp/mpv.sock playlist.m3u &

# Drive it from any shell
echo '{"command":["set_property","pause",true]}' | socat - /tmp/mpv.sock
echo '{"command":["seek",30]}'                   | socat - /tmp/mpv.sock
echo '{"command":["get_property","time-pos"]}'   | socat - /tmp/mpv.sock

# YouTube via yt-dlp, audio only, best quality
mpv --no-video --ytdl-format=bestaudio \
    'https://www.youtube.com/watch?v=...'

# A/B loop a section for transcription
mpv --ab-loop-a=00:01:23 --ab-loop-b=00:01:45 lecture.mp4
```

## Why it's interesting

The "player" market split a decade ago into GUI apps with
weak automation (VLC, IINA, QuickTime) and headless decoders
that don't render (raw `ffmpeg`/`ffplay`). `mpv` is the
seam between them: the same binary that drives a colour-
managed Vulkan pipeline on a workstation also runs as a
headless background daemon a shell script controls over a
Unix socket. Three things make it a load-bearing tool in a
terminal-resident workflow rather than just "another player":
(1) the JSON IPC means *any* automation that can write to a
socket can drive it — `tmux` keybinds, `i3` / `sway`
keybindings, `streamdeck` macros, voice-control daemons,
language-model agents queueing audiobook chapters; (2) the
Lua / JS in-process script API exposes the full property
graph (track list, sub delay, audio filters, OSD) so the
"add a feature" loop is "drop a 50-line `.lua` file in
`scripts/`" not "fork the player"; (3) the OpenGL / Vulkan
output and `libplacebo` integration give it the highest
playback quality available outside madVR — HDR-to-SDR
tone-mapping, EWA Lanczos scalers, ICC profile colour
management — at the cost of a steeper config surface than
"double-click the file". Pair with `yt-dlp` for streaming,
with `socat` / `jq` for IPC scripting, with `mpris2` /
`playerctl` for desktop integration; pick over [`vlc`](../vlc/)
when the player needs to be scripted from outside its own
process and over `ffplay` when you want a real UI / config
file / scriptable hotkeys instead of a debug viewer. Caveats
— the default config is intentionally minimal (most people
copy `~/.config/mpv/mpv.conf` from a curated dotfiles repo),
the GPL-2.0-or-later default build links GPL FFmpeg
(`-Dgpl=false` produces an LGPL-2.1-or-later binary if you
need to embed in a proprietary product), and the IPC socket
is unauthenticated (anyone with filesystem access to it can
control playback — keep it under a per-user runtime dir, not
`/tmp` shared).
