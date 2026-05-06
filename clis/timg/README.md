# timg

> **timg** — hzeller/timg, a terminal image **and video** viewer that
> renders pixels into a true-color terminal grid using half-block
> Unicode (`▀`) so each cell carries two pixels of vertical
> resolution, plus optional Sixel and Kitty graphics protocols when
> the host terminal supports them. Pinned to **v1.6.3**, GPL-2.0 —
> license file:
> [LICENSE](https://github.com/hzeller/timg/blob/main/LICENSE).

Source: <https://github.com/hzeller/timg>

## TL;DR

`timg image.png` paints the image into your terminal at the cell
grid's effective pixel resolution. `timg video.mp4` plays the video
inline at 30 fps using the same renderer, with audio left to the
host (timg only renders frames). `timg -g 80x40 *.jpg` lays out a
contact-sheet of every JPEG in the directory at fixed cell size.
`timg --grid=4x3 *.png` tiles 12 thumbnails into one terminal view.

The renderer is the differentiator: by default it uses Unicode
half-block (`▀`) characters with separate fore/background true-color
codes per cell, so each terminal row holds 2 pixels of vertical
resolution and the image effectively renders at `cols × (2·rows)`
pixels — twice the apparent resolution of a naive one-cell-per-pixel
renderer. With `-pk` (Kitty graphics protocol) or `-ps` (Sixel),
timg switches to true-pixel mode against terminals that support it
(kitty, WezTerm, foot, mlterm, xterm with Sixel, Konsole, Black Box,
etc.) for crisp 1:1 rendering with no half-block aliasing.

It also handles **animation** natively: animated GIFs, animated PNGs
(APNG), animated WebP, and any video format ffmpeg can decode all
play inline at the source frame rate, looping or one-shot. `--loops
N` controls iteration count, `-t SECONDS` caps duration, `-c COUNT`
caps frame count.

## What it actually solves

The "show me this image without leaving the terminal" problem has
three flavors:

1. **SSH to a remote box, want to look at the PNG you just generated**
   — copying it back to your laptop to open in Preview is friction.
   `timg ./output.png` shows it inline in the SSH session in under
   100 ms. No X11 forwarding, no scp.

2. **Iterating on a plot / chart / generated image in a script** —
   `python plot.py && timg out.png` is faster than alt-tabbing to a
   file manager and re-opening on every iteration.

3. **Browsing a directory of media** — `timg --grid=4x3 *.jpg`
   gives a contact sheet in one terminal pane, no GUI image viewer.

## Install

```bash
# macOS (Homebrew)
brew install timg

# Debian / Ubuntu
sudo apt install timg

# Build from source (CMake + ffmpeg + GraphicsMagick)
git clone https://github.com/hzeller/timg
cd timg && mkdir build && cd build
cmake ../ -DWITH_VIDEO_DECODING=On -DWITH_VIDEO_DEVICE=On
make && sudo make install
```

The Homebrew bottle is the easiest path on macOS — it pulls in
ffmpeg + GraphicsMagick + libwebp + libdeflate as deps so the full
codec set works out of the box.

## Why orthogonal to the existing zoo

The zoo already has terminal-graphics / image utilities, but timg
covers a different axis:

- [`chafa`](../chafa/) is the closest neighbor — also renders images
  into the terminal grid with multiple character backends (half-block,
  symbols, sixel, kitty, iTerm). chafa wins on the pure-ASCII /
  symbol-fitting renderer (it picks from a vocabulary of Unicode
  block / braille / box characters per cell, where timg sticks with
  the half-block primitive). **timg wins on video** — chafa does
  animated GIFs but timg's ffmpeg integration plays arbitrary video
  formats with proper frame timing. They coexist: chafa for the
  prettiest static image rendering on a terminal without graphics
  protocol support, timg for video and the half-block-or-pixel-protocol
  fast path. Pick by which flavor fits.
- [`viu`](../viu/) is the lighter Rust alternative — half-block + Sixel
  + Kitty + iTerm protocols, very fast for static images. **timg wins
  on video and APNG/WebP animation breadth**; viu wins on binary
  size and Rust portability.
- [`ascii-image-converter`](../ascii-image-converter/) is the
  classical-ASCII-art converter (densitometric character mapping with
  no color or with 256-color). Different aesthetic — timg targets
  *photographic fidelity* in true-color terminals, ascii-image-converter
  targets the retro look.
- [`hyfetch`](../hyfetch/) / [`fastfetch`](../fastfetch/) display
  small distro logos in the terminal — single-purpose vs timg's
  general-image / video viewer.
- [`mpv`](../mpv/) is a full-featured video player that *can* render
  to the terminal via libsixel, but mpv is a 100-MB-class media
  framework. timg is the small focused "play any video inline in a
  terminal" tool, ~3 MB binary plus ffmpeg.

In practice: keep timg for SSH + script-iteration workflows where
you want to glance at an image / video without opening a GUI app or
forwarding X11. Pair with chafa for the prettiest static-image
output on a terminal without graphics-protocol support.

## Pairs with

- [`fd`](../fd/) — `fd -e png | xargs timg --grid=4x3` to contact-sheet
  every PNG in a tree
- [`yazi`](../yazi/) / [`xplr`](../xplr/) — file managers that can use
  timg as a preview command for image / video columns
- [`ffmpeg`](https://ffmpeg.org) — already a transitive dep; explicitly
  available for re-encoding before piping to timg
- [`mpv`](../mpv/) — for full media-player ergonomics on the same host

## Caveats

- The half-block renderer's color fidelity depends on the terminal
  emulator's true-color (`COLORTERM=truecolor`) support — older
  256-color terminals will dither visibly.
- Video playback frame rate is bounded by terminal redraw speed —
  Apple Terminal.app caps lower than iTerm2 / WezTerm / kitty.
- ffmpeg is a runtime dep for video; without it, timg works for
  still images only.
- GPL-2.0 licensing means linking timg into a closed-source product
  isn't possible; that's a non-issue for CLI use, worth noting for
  libtimg embedding.
- `-V` (video) and `-pk` / `-ps` (graphics protocols) are mutually
  exclusive with some terminal-protocol combos; check `timg --help`
  on the specific terminal.
