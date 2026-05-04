# jpegoptim

> **Lossless (and optionally lossy-to-target-size) JPEG re-encoder
> that strips marker cruft and runs the libjpeg Huffman optimiser
> over every file in a directory in parallel.**
> Pinned to **v1.5.6**
> ([LICENSE](https://github.com/tjko/jpegoptim/blob/main/LICENSE),
> GPL-3.0).

Source: <https://github.com/tjko/jpegoptim>

## TL;DR

`jpegoptim` re-encodes JPEGs without re-decoding the DCT
coefficients: it reads the existing entropy-coded blocks, drops
ancillary markers (EXIF / IPTC / XMP / Adobe / JFIF / COM /
thumbnail) on request, and recomputes the optimal Huffman tables
that the original encoder probably did not bother to optimise. The
pixels come out **bit-identical** in the lossless mode (default);
files typically shrink **5–15%** for one-shot consumer-camera
JPEGs and **20–30%** for "save for web" JPEGs that shipped with
default Huffman tables and embedded thumbnails. The lossy mode
(`--size` / `--max`) does re-decode and re-quantise to hit a
target file size or maximum quality, useful when a CDN imposes a
per-image byte budget. A `--workers=N` flag (1.5.0+) parallelises
across CPU cores so a 10 000-image directory finishes in seconds.

## Install

```bash
# macOS / Linux Homebrew
brew install jpegoptim          # 1.5.6

# Debian / Ubuntu
apt install jpegoptim

# Fedora / RHEL
dnf install jpegoptim

# Arch
pacman -S jpegoptim

# from source (libjpeg / libjpeg-turbo / mozjpeg)
git clone https://github.com/tjko/jpegoptim.git
cd jpegoptim && git checkout v1.5.6
./configure && make && sudo make install

# verify
jpegoptim --version             # jpegoptim v1.5.6
```

`jpegoptim` is a small C program (~80 KB binary) with one
runtime dep (`libjpeg`, `libjpeg-turbo`, or `mozjpeg` — the same
binary works against any). No Node, no Python, no GPU. Linking
against `mozjpeg` instead of stock `libjpeg-turbo` enables
trellis quantisation at the cost of CPU; useful for one-shot
"hero" assets, overkill for batch.

## License

GPL-3.0 — see
[LICENSE](https://github.com/tjko/jpegoptim/blob/main/LICENSE).
Running `jpegoptim` as a child process in a build pipeline is
non-viral (the JPEGs you ship are not derived works of
`jpegoptim`'s source). Statically linking `jpegoptim` *into* a
closed-source binary is the case GPL would force the binary
under GPL itself; if that's the goal, switch to libjpeg-turbo's
own `cjpeg`/`jpegtran` (BSD-style).

## One Concrete Example

```bash
# 1. lossless: re-encode in place, keep all markers, ~5-15% shrink
jpegoptim photo.jpg
# photo.jpg 3.2M → 2.7M (-15.6%) [OK]

# 2. lossless + strip everything (EXIF / thumbnail / IPTC / XMP / ICC)
jpegoptim --strip-all photo.jpg
# photo.jpg 3.2M → 2.5M (-21.9%) [OK]

# 3. preserve EXIF (so cameras / Lightroom / Photos still see capture metadata)
jpegoptim --strip-all --keep-exif photo.jpg

# 4. force progressive (better perceived load on slow connections)
jpegoptim --all-progressive photo.jpg

# 5. lossy — target ≤200 KiB (re-quantises to hit the budget)
jpegoptim --size=200k --strip-all photo.jpg

# 6. lossy — target maximum quality 85 (capped if input is higher)
jpegoptim --max=85 --strip-all photo.jpg

# 7. recurse a directory in parallel across all CPUs
jpegoptim --strip-all --all-progressive --workers=8 -r photos/

# 8. dry-run — print expected savings without writing
jpegoptim --noaction --strip-all photo.jpg
# photo.jpg 3.2M → 2.5M (-21.9%) [OK] (NOT WRITTEN)

# 9. read file list from stdin (composes with find / git ls-files)
git ls-files '*.jpg' | jpegoptim --files-stdin --strip-all --workers=8

# 10. CSV output for a CI savings report
jpegoptim --csv --strip-all -r photos/ > savings.csv
# filename,size_before,size_after,size_diff,quality,markers,result

# 11. CI gate — fail if any JPEG didn't shrink to within 10% of target
jpegoptim --size=300k --noaction --strip-all -r photos/ > /tmp/sizes.txt
awk '/->/ { ... }' /tmp/sizes.txt   # parse and gate as needed
```

## Niche It Fills

**Lossless re-encoding of JPEGs in a build pipeline + a clean
"shrink-to-target-size" lossy fallback** — the JPEG-shaped twin
of [`pngquant`](../pngquant/) (which owns the PNG slot) and
[`oxipng`](../oxipng/) (which owns the lossless-PNG slot). Most
JPEGs in the wild ship with default Huffman tables, embedded
thumbnails, full EXIF + GPS, and editor-specific APP markers
nobody downstream cares about; `jpegoptim` strips that
deterministically and recomputes the entropy coding without
re-quantising the coefficients, so the visual output is provably
identical. The result is the cheap, no-quality-loss pass every
asset pipeline should run before serving JPEGs, with the lossy
`--size` mode as the escape hatch for "must fit in 200 KiB".

## Why use it

1. **Lossless by default — pixels are bit-identical to the
   input.** A `cmp` of the decoded raster before and after a
   default-mode `jpegoptim` run returns identical buffers; the
   savings come purely from better Huffman tables and dropped
   markers, not from re-quantising. This is the key safety
   property that lets you wire it into CI without a perceptual
   diff step.
2. **`--strip-*` is granular.** `--strip-all` drops every
   ancillary marker; `--keep-exif` / `--keep-iptc` / `--keep-icc`
   / `--keep-xmp` / `--keep-jfif` re-add the ones you want. So
   "strip everything except colour profile" (so the JPEG keeps
   its sRGB tag and renders correctly on wide-gamut displays) is
   `--strip-all --keep-icc` — one flag, deterministic, no
   metadata-toolchain detour through `exiftool`.
3. **`--size` / `--max` give you a CDN-friendly hard budget.**
   When the constraint is "every JPEG under `assets/hero/` must
   be ≤200 KiB to fit a per-image budget", `jpegoptim
   --size=200k --strip-all -r assets/hero/` does exactly that:
   re-encodes only the files over budget, drops quality just
   enough to fit, and leaves files already under budget
   untouched. No "compute target quality per file" Python
   wrapper required.
4. **`--workers=N` parallelises the directory walk.** A 10 000-
   JPEG asset tree on an 8-core box runs ~7× faster with
   `--workers=8` than the single-threaded default. There is no
   shared encoder state across files so the scaling is close to
   linear up to physical core count.

For an LLM-CLI workflow, `--csv` produces structured per-file
output (`filename, size_before, size_after, size_diff, quality,
markers, result`) and `--noaction` is a true dry-run mode — an
agent can plan a savings report and present it for approval
before any byte of the working tree changes.

## Vs Already Cataloged

- **Vs [`pngquant`](../pngquant/) and [`oxipng`](../oxipng/):**
  different format (lossy / lossless raster). The three sit at
  orthogonal slots in an asset pipeline: `pngquant` owns lossy
  PNG-with-alpha (palette quantisation), `oxipng` owns lossless
  PNG (deflate / zopfli re-encode), `jpegoptim` owns JPEG (drop
  markers + Huffman re-encode, with optional re-quantise to
  budget). A typical static-site `Makefile` runs all three over
  their respective globs without conflict.
- **Vs [`svgo`](../svgo/):** different format (vector). `svgo`
  optimises SVG XML AST; `jpegoptim` optimises JPEG entropy
  coding. They never touch the same file.
- **Vs [`gifski`](../gifski/) / [`gifsicle`](../gifsicle/):**
  different format (animated GIF / PNG-sequence-to-GIF).
- **Vs `cjpeg` / `jpegtran` from libjpeg-turbo / mozjpeg:**
  `jpegtran` is the raw lossless-transform building block
  (Huffman re-optimisation, marker stripping, rotation) that
  `jpegoptim` wraps; `cjpeg` is the encoder used to *re-create*
  a JPEG from a decoded raster (the lossy mode here). `jpegoptim`
  is the directory-friendly, batch-mode, CSV-emitting,
  `--size`-aware façade — pick the underlying tools when you
  need a single in-line transform inside a larger C/C++ image
  pipeline; pick `jpegoptim` for everything that walks a
  directory.
- **Vs re-encoding to WebP / AVIF / JPEG-XL:** the modern
  formats are 20–40% smaller again at equal perceptual quality
  but require either format negotiation (`<picture>` /
  `Accept: image/avif`) or a CDN that does it for you.
  `jpegoptim` ships JPEGs that every browser, OS, mail client,
  and PDF renderer back to 1995 supports — pick it when "must
  be a `.jpg` that opens everywhere" is a hard requirement and
  the modern formats when it isn't.

## Caveats

- **GPL-3.0.** Running the CLI in CI is fine. Vendoring its
  source into a closed-source binary forces that binary under
  GPL — for embedding, use libjpeg-turbo's `jpegtran` directly
  (BSD) and reimplement the directory walk + marker policy
  yourself.
- **Lossy mode (`--size` / `--max`) re-decodes the JPEG.** This
  means a second generational loss compared to the original;
  every subsequent re-run of `jpegoptim --size=…` on the same
  file will lose more quality. Cure: always run `--size` /
  `--max` against the *original* asset, not against an output
  of a previous `jpegoptim` lossy pass. Default lossless mode
  is safe to re-run forever (idempotent after the first call).
- **No knowledge of perceptual metrics (SSIM / Butteraugli).**
  `--max=85` is the libjpeg quality scale, which is *not*
  perceptually uniform — JPEG quality 85 of one image looks
  very different from quality 85 of another. For a perceptual-
  quality-targeted re-encode, reach for `cjpeg` from
  [`mozjpeg`] with `-quality` plus a downstream Butteraugli /
  SSIMULACRA2 check.
- **`-r` follows symlinks unless `--no-symlinks` is set.**
  Worth knowing if your asset tree symlinks into another
  worktree (e.g. submodules of an `assets/` repo).
- **Doesn't touch `<img srcset>` / responsive variants.**
  `jpegoptim` operates on individual files; if your build needs
  to also generate `image-1x.jpg` / `image-2x.jpg` / `image-3x.jpg`
  variants, that's a separate resize step (`vips`, `ImageMagick`,
  `sharp`); `jpegoptim` is the post-step that runs on each
  variant.
- **Default behaviour preserves all markers.** A surprising
  number of users run `jpegoptim *.jpg` and are confused that
  the savings are smaller than blog posts claim — those posts
  almost always assume `--strip-all`. Decide your marker policy
  first, then commit it to the build script; never run
  `jpegoptim` without an explicit `--strip-*` choice in CI.
