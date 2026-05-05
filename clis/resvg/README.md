# resvg

> **Pure-Rust SVG rendering CLI** that rasterizes (or re-emits)
> SVG files to PNG/PDF without pulling in a browser, Cairo, or
> librsvg. The reference implementation of a strict subset of
> the SVG 1.1 / 2 static spec, by the linebender project. Pinned
> to **v0.47.0** (SPDX: `Apache-2.0`,
> [LICENSE-APACHE](https://github.com/linebender/resvg/blob/main/LICENSE-APACHE)).

Source: <https://github.com/linebender/resvg>

## TL;DR

`resvg` is the right answer to "I have an SVG and I want a PNG,
deterministically, on a headless box, without installing
ImageMagick or running headless Chrome." Single Rust binary.
Reads an SVG, rasterizes it to PNG (or writes a flattened SVG /
PDF), with explicit width/height/DPI/zoom controls and a
documented spec-coverage matrix so you know up front what it
does and does not support.

## Install

```bash
# Homebrew (macOS / Linux)
brew install resvg

# Cargo (builds the `resvg` CLI from the workspace)
cargo install resvg --locked

# Pre-built release binary
curl -Lo resvg.tar.gz "https://github.com/linebender/resvg/releases/download/v0.47.0/resvg-aarch64-apple-darwin.tar.gz"
tar xf resvg.tar.gz
sudo install resvg /usr/local/bin/

# verify
resvg --version    # resvg 0.47.0
```

## License

Apache-2.0 (also available under MIT) — see
[LICENSE-APACHE](https://github.com/linebender/resvg/blob/main/LICENSE-APACHE)
and
[LICENSE-MIT](https://github.com/linebender/resvg/blob/main/LICENSE-MIT).

## One Concrete Example

```bash
# 1. straight rasterize: SVG -> PNG at intrinsic size
resvg input.svg out.png

# 2. force a width (height auto-scales to preserve aspect ratio)
resvg --width 1024 input.svg out.png

# 3. set both dimensions explicitly
resvg --width 1024 --height 1024 input.svg out.png

# 4. zoom 2x for retina assets
resvg --zoom 2 logo.svg logo@2x.png

# 5. set background (default is transparent)
resvg --background '#ffffff' input.svg out.png

# 6. point at a custom font directory for text rendering
resvg --use-fonts-dir ./fonts input.svg out.png

# 7. dump what resvg actually parsed (post-flattening, post-CSS)
resvg --dump-svg flattened.svg input.svg out.png

# 8. read SVG from stdin, write PNG to stdout
cat input.svg | resvg - -
```

## Niche It Fills

**Headless, reproducible SVG -> PNG conversion.** The
production-grade alternatives are: (a) ImageMagick + librsvg
(huge dependency surface, fragile across distros), (b) headless
Chrome (multi-hundred-MB binary, fonts vary by platform), or
(c) Inkscape CLI (heavyweight, slow startup). `resvg` is a
single ~10 MB Rust binary with no system dependencies and a
published spec-coverage matrix so output is bit-stable across
machines.

## Why use it

1. **Deterministic output.** Same SVG + same `resvg` version =
   same PNG bytes, on Linux, macOS, CI, any arch. Critical for
   visual regression tests.
2. **No system libraries.** Static Rust binary, no Cairo, no
   librsvg, no Pango. Drop into a scratch container and it
   works.
3. **Spec-coverage is explicit.** The repo ships a coverage
   table — if your SVG uses `<filter>` with a feature `resvg`
   does not support, you know up front instead of getting
   silently wrong output.
4. **Designed for batch.** No GUI surface, no interactive
   prompts, friendly to `find ... -exec` and Make rules.
5. **Backs other Rust ecosystems.** `resvg` is the SVG
   renderer the Rust GUI / typesetting world (e.g. `typst`)
   pulls in, so behavior is well-exercised.

## Vs Already Cataloged

- **Vs ImageMagick / `magick`:** ImageMagick will rasterize SVG
  but only by shelling out to librsvg or its built-in (very
  partial) renderer. `resvg` is a single binary with predictable
  behavior.
- **Vs [`oxipng`](../oxipng/) / [`pngquant`](../pngquant/):**
  Those operate on existing PNGs (lossless / lossy compression).
  `resvg` is upstream of them — *produce* the PNG, then optimize.
- **Vs [`svgo`](../svgo/):** `svgo` shrinks an SVG (still SVG
  out). `resvg` rasterizes (PNG out). Different stage.
- **Vs [`chafa`](../chafa/) / [`viu`](../viu/):** Those render
  images *to the terminal*. `resvg` writes image files for
  pipelines, build outputs, and asset packaging.

## Caveats

- **Static SVG only.** No SMIL animation, no `<script>`,
  no interactivity. By design.
- **Filter coverage is partial.** Complex `<filter>` graphs
  (especially `feColorMatrix` chains and Gaussian blur edge
  cases) may render differently from a browser. Check the
  coverage table.
- **Fonts must be findable.** Text uses the system font stack;
  in a minimal container set `--use-fonts-dir` or vendor the
  fonts you reference.
- **PDF output is utility-grade.** Fine for "embed this SVG in
  a PDF report"; not a substitute for a PDF typesetting engine.
