# cava

> **Console-based Audio Visualizer for ALSA / PulseAudio /
> PipeWire / sndio / portaudio / OSS / JACK / macOS CoreAudio /
> Windows WASAPI** — a real-time spectrum-analyser bar graph
> rendered with Unicode block characters in any terminal,
> driven by an FFT over the system audio loopback (or any
> chosen input device), with configurable bar count, sensitivity
> auto-adjust, gradient colour modes, mirrored / mono / stereo
> layouts, smoothing constants (integral + monstercat), and a
> raw-stdout mode that lets other programs consume the bar
> heights as a data stream. Pinned to **0.10.7** (commit
> `bbfbd0566bd3ced82b3d2a195d9c343b31b8b419`,
> [LICENSE](https://github.com/karlstav/cava/blob/master/LICENSE),
> MIT).

Source: <https://github.com/karlstav/cava>

## TL;DR

`cava` reads PCM samples off your system's audio loopback (or
any explicitly chosen input), runs an FFT to produce frequency-
bin amplitudes, smooths them, and draws a vertical bar graph
across the terminal width using Unicode 1/8-block characters
(`▁▂▃▄▅▆▇█`) so the bars are sub-character-height accurate. The
result is a music visualiser that runs in any 24-bit-colour
terminal with no GUI, no X server, no GPU — useful as a desktop
toy on a tiling WM, as the "now playing" widget in a tmux pane,
as a stage prop for live coding / streaming, or (via
`output.method = raw` and a config-driven stdout pipe) as a
bar-height data feed for `conky` / `polybar` / a custom LED-bar
controller / a Pi rack-light. One C binary, builds against any
of ALSA / PulseAudio / PipeWire / JACK / sndio / portaudio /
OSS on Linux + BSD, CoreAudio on macOS, WASAPI on Windows, and
SDL for raw output to a window when you want it.

## Install

```bash
# Homebrew (macOS — uses portaudio backend)
brew install cava

# Debian / Ubuntu (PulseAudio + ALSA backends)
sudo apt install cava

# Arch / Manjaro (PipeWire + PulseAudio + ALSA)
sudo pacman -S cava

# Fedora
sudo dnf install cava

# Nix
nix-env -iA nixpkgs.cava

# from source (full backend matrix)
git clone https://github.com/karlstav/cava
cd cava
./autogen.sh && ./configure && make
sudo make install

# verify
cava -v       # C.A.V.A. 0.10.7
```

On macOS `cava` reads from default CoreAudio input — to
visualise *system audio* (not the microphone), install
[BlackHole](https://github.com/ExistentialAudio/BlackHole) (a
free virtual loopback device), set it as system output, and
point `cava` at it via `~/.config/cava/config`:

```ini
[input]
method = portaudio
source = "BlackHole 2ch"
```

On PipeWire-based Linux, the default config "just works" against
`pipewire-pulse` — no extra setup.

## License

MIT — see
[LICENSE](https://github.com/karlstav/cava/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. just run it (default config = PulseAudio / PipeWire system loopback)
cava
#    bars dance across the terminal width while music plays

# 2. specific device + custom config
cat > ~/.config/cava/config <<'EOF'
[general]
bars = 64
framerate = 60
sensitivity = 100
autosens = 1
[color]
gradient = 1
gradient_color_1 = '#59cc33'
gradient_color_2 = '#80ff00'
gradient_color_3 = '#ffee00'
gradient_color_4 = '#ff7700'
gradient_color_5 = '#ff0000'
[smoothing]
monstercat = 1
integral = 77
EOF
cava

# 3. embed in a tmux status pane (3 rows, low refresh, mono)
cava -p <(cat <<'EOF'
[general]
bars = 40
framerate = 30
[output]
mode = normal
channels = mono
EOF
)

# 4. raw mode — pipe bar heights to another program
cat > /tmp/cava-raw.conf <<'EOF'
[general]
bars = 16
[output]
method = raw
raw_target = /dev/stdout
data_format = ascii
ascii_max_range = 100
EOF
cava -p /tmp/cava-raw.conf | awk -F';' '{print "peak:", $1, "/100"}'
#    each line = current frame's 16 bar heights, semicolon-delimited

# 5. SDL output (real graphical window) — when terminal width is too narrow
cava -p <(echo -e '[output]\nmethod = sdl\nsdl_width = 1280\nsdl_height = 240')
```

## Niche It Fills

**A real-time audio FFT visualiser that runs in a terminal.**
Every other "audio visualiser" is either a GUI app
(`projectM`, Spotify's built-in, Apple Music's), a media-player
plugin (foobar2000, `mpv --audio-display`), or a heavyweight
desktop widget (Rainmeter, Cairo Dock). `cava` is none of those:
it is a 60-fps spectrum analyser in a tmux pane, an SSH
session, a TTY without X, or a Raspberry Pi headless boot —
anywhere there is a terminal and an audio source. The raw-mode
data-feed turns it into a generic FFT-bars producer that other
programs (status bars, LED controllers, tmux widgets, OBS scenes)
can consume.

## Why use it

Three reasons it earns the disk space versus the GUI options:

1. **Backend matrix is huge.** ALSA, PulseAudio, PipeWire,
   JACK, sndio, portaudio, OSS, CoreAudio, WASAPI — pick the
   audio stack you actually have, no matter which OS / distro,
   and `cava` builds against it. Most GUI visualisers are
   PipeWire-or-die or PulseAudio-only and need rewriting when
   the audio stack changes.
2. **Raw-mode = data-feed for anything.** `output.method = raw`
   emits frame-by-frame bar heights to a file / fifo / stdout
   in `ascii` or `binary` format with configurable bar count
   and value range. Wire it to `conky`, `polybar`, `i3blocks`,
   a serial port driving an LED bar, an OBS browser source —
   `cava` becomes a generic FFT-bars producer.
3. **Headless / SSH / TTY friendly.** No X / Wayland / GUI
   dependency by default. Lives happily in a tmux pane on a
   remote box, on a TTY with no display server, on a Pi
   driving a small terminal-only display. Add SDL output for
   a graphical window only when the terminal is the wrong
   shape.

For an LLM-CLI workflow it is mostly desk candy — but a tmux
status pane that shows a 16-bar spectrum of the current
playback gives a quick "is the music actually playing?" tell
when headphones are flaky and is a more pleasant idle screen
than `htop`.

## Vs Already Cataloged

- **Vs [`mpv`](../mpv/):** complementary — `mpv` is the
  player; `cava` visualises the audio (which can come from
  `mpv` or any other source). Pipe `mpv` output via a JACK or
  PipeWire loopback to give `cava` the exact stream.
- **Vs [`peaclock`](../peaclock/) /
  [`pipes-sh`](../pipes-sh/) / [`cmatrix`-style toys](../patat/):**
  same "terminal eye candy" category, different content
  (clock / animated pipes / spectrum analyser). `cava` is the
  audio-reactive one.
- **Vs `projectM` / Spotify-built-in / `mpv --audio-display`:**
  GUI-only or player-bound; `cava` runs in a terminal against
  any audio source the OS exposes.
- **Vs [`s-tui`](../s-tui/) / [`bottom`](../bottom/) /
  [`btop`](../btop/):** same "live TUI graph" category but
  unrelated content (CPU thermals / system stats vs audio
  spectrum). They share a tmux-pane-friendly aesthetic and
  pair well in a status board.

## Caveats

- **Needs a loopback to visualise *system* audio on macOS /
  Windows.** CoreAudio and WASAPI do not expose default-output
  as an input stream — install BlackHole (macOS) or VB-Cable
  (Windows) and route system audio through it. Linux PipeWire
  / PulseAudio do this natively via the monitor source.
- **Truecolour terminal recommended.** Gradient mode produces
  16M-colour gradients; on a 256-colour terminal it dithers
  badly. Ghostty / Kitty / iTerm2 / Wezterm / Alacritty are
  fine; macOS `Terminal.app` is not.
- **CPU usage scales with framerate × bar count × terminal
  width.** Default 60 fps × 64 bars on a 4K terminal pulls a
  measurable percent of one core. Drop `framerate` to 30 and
  `bars` to 40 for a tmux-pane widget.
- **No audio capture, no music identification.** It only
  visualises — does not record, does not save spectrograms to
  disk (use `sox -n spectrogram` for that), does not name the
  song (`mpv` / `playerctl metadata` does that).
- **Build-time backend choice.** A given binary supports the
  backends it was *compiled* with. The Homebrew bottle ships
  portaudio; the apt package ships PulseAudio + ALSA; build
  from source if you need PipeWire-native or JACK on macOS.
- **Not for accessibility-critical use.** It is decorative.
  Do not read clinical decisions off the bars.
