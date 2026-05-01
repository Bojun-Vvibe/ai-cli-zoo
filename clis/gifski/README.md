# gifski

> **`pngquant`-author's GIF encoder that does per-frame neural-quantised
> 256-colour palettes with temporal dithering, producing GIFs that
> look 2–3x sharper than `ffmpeg -f gif` at the same byte budget — the
> last-mile encoder for every other recorder in the zoo.** Pinned to
> **v1.34.0**, AGPL-3.0
> ([LICENSE](https://github.com/ImageOptim/gifski/blob/main/LICENSE)).

- **Repo:** https://github.com/ImageOptim/gifski
- **Latest version:** v1.34.0
- **License:** AGPL-3.0 (`LICENSE` at repo root, SPDX `AGPL-3.0-or-later`;
  commercial dual-license available from the author)
- **Category:** `media-encoding` / `docs-tooling`
- **Language:** Rust

## What it does

`gifski` takes a sequence of PNG / APNG / MP4 / WebM frames and emits
a GIF whose palette is computed *per frame* by the same `pngquant`
neural quantiser the author wrote, then temporally dithered so the
256-colour limit doesn't become a visible flicker between frames. The
practical consequence is that a 1200×800 terminal recording at 24 fps
fits in 1–2 MB at a quality the standard `ffmpeg -vf
"palettegen,paletteuse"` two-pass cannot reach below 4–5 MB. Inputs
can be a glob of PNGs (`gifski -o out.gif frame*.png`), a single
video file (`gifski -o out.gif demo.mp4`), or stdin via `--`. Output
controls are deliberately small: `--quality 1..100`, `--fps`,
`--width`, `--height`, `--lossy-quality`, `--repeat`, plus
`--motion-quality` to trade temporal dithering for file size when the
source is mostly static text.

## Install

```sh
# macOS / Linux via Homebrew
brew install gifski

# Cross-platform via cargo
cargo install gifski --version 1.34.0

# Pre-built binaries on every release page
# https://github.com/ImageOptim/gifski/releases/tag/1.34.0
```

## Usage

```sh
# Encode a vhs / agg recording into a tight GIF
gifski -o demo.gif --fps 24 --width 1200 --quality 90 frames/*.png

# Re-encode an existing MP4 (drop frames, downscale, cap palette)
gifski -o demo.gif --fps 12 --width 800 demo.mp4
```

## Use when

- Your README GIF is over 5 MB and GitHub's image proxy is starting
  to compress it into mush.
- You're piping frames out of [`vhs`](../vhs/), [`agg`](../agg/),
  [`asciinema`](../asciinema/), or [`t-rec`](../t-rec/) and want a
  smaller, sharper final artifact than those tools' built-in encoders.
- You need deterministic, reproducible GIF bytes from the same input
  frames (encoder is single-threaded-stable on `--quality` ties).

## Skip when

- AGPL-3.0 is incompatible with your downstream distribution (use
  `gifsicle` MIT instead, lower visual quality).
- You need WebP / AVIF / animated PNG output — `gifski` only emits
  GIF; pair with `ffmpeg` for other containers.
- The recording is already a GIF and you just need to optimise it
  losslessly — use `gifsicle -O3` for that path.

## Comparison to nearest neighbours

- **vs [`vhs`](../vhs/):** `vhs` *records* a deterministic terminal
  session and ships with its own ffmpeg-based encoder; `gifski` is
  the higher-quality replacement when you keep `vhs --output frames`
  and re-encode externally.
- **vs [`agg`](../agg/):** `agg` converts asciinema casts to GIF
  using a built-in renderer that's fast but coarse on colour-rich
  themes; `gifski` does a better palette job at the cost of needing
  an intermediate frame dump.
- **vs `ffmpeg -f gif`:** ffmpeg's two-pass `palettegen,paletteuse`
  is the standard baseline; `gifski` reliably wins on byte-size at
  matched perceptual quality, especially for screen content with
  anti-aliased text.
