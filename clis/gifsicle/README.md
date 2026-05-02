# gifsicle

- **Upstream:** https://github.com/kohler/gifsicle
- **Version:** v1.96 (tagged 2025-02-26)
- **License:** GPL-2.0 (`COPYING`)
- **Language:** C

## What it is

`gifsicle` is Eddie Kohler's long-running command-line tool for creating, manipulating, and
optimizing GIF images. It reads one or more GIFs and edits their structure: combine separate frames
into an animation (`--delay`, `--loopcount`), explode an animation back into frames (`--explode`),
crop / resize / rotate / dither, swap or remap color tables (`--colors 64 --dither`), strip
metadata, and — the headline feature — losslessly optimize an existing animation by detecting
inter-frame redundancy and shipping each frame as a minimal partial-frame disposal patch
(`-O3` is the standard "shrink this animation" knob, often 30-70 % smaller with no visible
quality change). Companion binary `gifview` displays the result in an X11 window.

## Why an AI/CLI user might pick it

GIF is still the universal "drop into any chat / issue tracker / docs site without an embed
player" animation format, and `gifsicle` is the canonical tool for the post-processing tail of
every screen-recording / terminal-cast / model-output pipeline: `vhs` / `t-rec` / `asciinema`-to-GIF
exports, `ffmpeg`-rendered training-curve animations, screenshot-comparison sequences, and
generated GIFs from image models all go through `gifsicle -O3 --colors 128` to turn a 4 MB
animation into one small enough to paste into a PR description. It is in every distro repo, the
binary is ~150 KB, and the flag set is stable across decades — install it once and never think
about it again.

## Install

```sh
brew install gifsicle
```

## Example

```sh
# Combine PNG-extracted frames into a 10 fps looping GIF, then optimize aggressively
gifsicle --delay=10 --loopcount=0 frame-*.gif -O3 --colors 128 -o demo.gif
```
