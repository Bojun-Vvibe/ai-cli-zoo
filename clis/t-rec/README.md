# t-rec

## What it does
A **terminal recorder that emits an animated GIF directly** — no
intermediate cast format, no separate render step, no headless browser.
Point it at a terminal window (`t-rec` auto-detects the focused
terminal on Linux / macOS, or you can name an `--win-id`), it grabs
frames at a configurable rate (default 4 fps with adaptive
deduplication), shells out to native screen-capture APIs (X11 +
Wayland + Quartz on macOS) so the recording is the *real* terminal
including font rendering, ligatures, true colour, and any GPU-accelerated
effects (kitty graphics, sixel images, image previews from `chafa`),
and on `Ctrl+D` writes a single `.gif` to disk plus optional `.mp4`.
Generates one frame per `Enter`-press by default (the "natural typing
cadence" mode) so the resulting GIF is paced like a real demo instead
of a uniform film strip, and ships an idle-frame compactor that drops
duplicate frames between commands so a 60-second recording with 8
seconds of typing produces a 12-second GIF.

## Why it's interesting
Different shape from `asciinema` (records a `.cast` JSON of typed
keystrokes — needs `agg` / `asciinema-player` to render, and the
playback is a *re-execution* of escape sequences in a JS terminal so
GPU-only features like kitty graphics, sixel, or font ligatures are
silently dropped), from `vhs` (script-driven, deterministic, but
**not** a recorder of an interactive session — you author a `.tape`
file in advance), from `terminalizer` (Node + Electron, ~300 MB
install, slow), from screen-recording the whole desktop with QuickTime
/ OBS / `wf-recorder` (captures the chrome, mouse cursor, notification
banners — you spend more time cropping than recording). t-rec is the
*one-binary terminal-only screen-grab to GIF* shape: pick it
specifically when you want a README demo of an interactive TUI session
where the *visual* output (sixel images, kitty graphics, ligatures,
true-colour theme) matters and you don't want to author a `vhs` tape.
Do **not** pick it for fully reproducible recordings (use `vhs`), for
script-friendly cast files that diff cleanly in PRs (use `asciinema`),
or for full-desktop screen recording (use OBS / QuickTime).

## Niche category
Native-capture terminal-to-GIF recorder — real screen grab of an
interactive terminal session with idle-frame compaction and per-Enter
pacing.

## Repo
https://github.com/sassman/t-rec-rs

## Version pinned
`v0.8.2` (latest tagged release, published 2025-12-19)

## License
- SPDX: `GPL-3.0-only`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS — Quartz capture path)
brew install t-rec

# Cargo (Linux + macOS; Linux build needs imagemagick + libx11-dev)
cargo install t-rec

# Arch Linux
pacman -S t-rec

# Nix
nix-env -iA nixpkgs.t-rec

# Prebuilt binaries
# https://github.com/sassman/t-rec-rs/releases/tag/v0.8.2
```

## Usage examples
```sh
# Default: auto-detect focused terminal, record until Ctrl+D, write
# ./t-rec.gif at 4 fps with idle-frame deduplication
t-rec

# Natural typing cadence — emit one frame per Enter press so the GIF
# is paced like a real demo, not a uniform film strip
t-rec --natural

# Higher frame rate for animated TUIs (btop / lazygit / nvim scrolling)
t-rec --frame-rate 8

# Also emit an mp4 alongside the gif (uses ffmpeg)
t-rec --video

# Record a specific window by id (X11 / macOS) instead of the focused one
t-rec --win-id 0x4400003

# Pause / resume during recording with Ctrl+T (handy for "type a long
# command, pause, switch contexts in your head, resume")
t-rec --start-pause

# Output to an explicit path (default is ./t-rec.gif)
t-rec --output demo-2026-04 --quiet
```
