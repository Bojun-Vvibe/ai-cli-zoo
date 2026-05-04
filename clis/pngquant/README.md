# pngquant

> **Lossy 24-bit-PNG → 8-bit-palette quantiser that preserves
> alpha and typically halves file size with no visible
> degradation.**
> Pinned to **v3.0.3**
> ([COPYRIGHT](https://github.com/kornelski/pngquant/blob/main/COPYRIGHT),
> GPL-3.0-or-later or commercial).

Source: <https://github.com/kornelski/pngquant>

## TL;DR

`pngquant` reads a 24- or 32-bit-RGBA PNG, runs a perceptual
median-cut quantiser ([`libimagequant`]) to choose an optimal
≤256-colour palette, applies Floyd–Steinberg dithering with
gamma-correct premultiplied-alpha math, and writes a fully
standards-compliant **palette PNG with alpha channel** (PNG-8 with
`tRNS`). Browsers, OS image viewers, mobile platforms, and PDF
renderers all support this — there is no compatibility cost. The
typical screenshot, photo, or asset PNG shrinks **60–80%** with no
visible diff at normal viewing distance, and even side-by-side
inspection at 100% zoom usually requires looking for the dithering
on smooth gradients to spot it. The `--quality min-max` flag is
the headline knob: it tells `pngquant` the acceptable quality
band, picks the smallest palette that hits the upper bound, and
exits non-zero (`99`) without writing the file if it cannot meet
the lower bound — making it directly usable as a CI gate.

## Install

```bash
# macOS / Linux Homebrew
brew install pngquant            # 3.0.3

# Debian / Ubuntu
apt install pngquant             # repo version varies (often 2.x)

# Fedora / RHEL
dnf install pngquant

# Arch
pacman -S pngquant

# from source (C99 + libpng + OpenMP)
git clone https://github.com/kornelski/pngquant.git
cd pngquant && git checkout 3.0.3
make && sudo make install

# verify
pngquant --version               # 3.0.3 (Oct 2023)
```

`pngquant` is a single small C binary (~150 KB) with one runtime
dep (`libpng`); the `libimagequant` engine is statically linked.
No Node, no Python, no GPU.

## License

Dual-licensed:

- **GPL-3.0-or-later** for open-source use, with an additional
  copyright notice that must be retained on the older parts of
  the code — see
  [COPYRIGHT](https://github.com/kornelski/pngquant/blob/main/COPYRIGHT).
- A separate **commercial license** (sold via Super Source) for
  embedding into closed-source products / App Store binaries
  where GPL is incompatible.

For CLI use in a build pipeline (run `pngquant` as a child
process to optimise files you ship) the GPL is non-viral — your
asset pipeline isn't a derived work of `pngquant`. Linking
`libimagequant` *into* a closed-source binary is what the
commercial license addresses.

## One Concrete Example

```bash
# 1. one file → writes screenshot-fs8.png next to it (default suffix)
pngquant screenshot.png
# screenshot.png: 1.4 MiB  →  screenshot-fs8.png: 412 KiB (-71%)

# 2. quality band — accept 65–80 (JPEG-like scale), refuse if <65
pngquant --quality=65-80 screenshot.png
# exit code 99 if no palette hits ≥65 → CI-detectable

# 3. overwrite in place (use with caution; combine with --skip-if-larger)
pngquant --ext=.png --force --skip-if-larger screenshot.png

# 4. recurse a directory of icons / screenshots
find assets -name '*.png' -exec pngquant --ext=.png --force {} +

# 5. stdin → stdout — drop into any pipe
cat icon.png | pngquant --quality=70-90 - > icon.min.png

# 6. strip metadata (EXIF / colour profiles) for max shrink
pngquant --strip --quality=70-85 -o icon.min.png icon.png

# 7. speed/quality knob — speed=1 is slowest+best, default 4, 11 is fastest
pngquant --speed=1 --quality=80-95 hero.png

# 8. CI gate — every PNG under assets/ must shrink to <500 KiB at q≥70
for f in assets/*.png; do
  pngquant --quality=70-95 --skip-if-larger -o /tmp/q.png "$f" || {
    echo "$f failed quality gate"; exit 1;
  }
  test "$(stat -f%z /tmp/q.png)" -lt 512000 || {
    echo "$f over budget"; exit 1;
  }
done
```

## Niche It Fills

**Lossy palette quantisation of PNGs that still need an alpha
channel** — exactly the case browsers, design systems, and mobile
apps hit constantly and that lossless tools cannot help with.
Lossless PNG optimisers ([`oxipng`](../oxipng/), `zopflipng`,
`pngcrush`) only re-encode the existing pixels; they shave 5–15%
off a typical PNG. `pngquant` reduces colour depth — the lossy
step that actually hits 60–80% reduction — and is the right first
pass before piping into an `oxipng` lossless re-encode for the
last few percent. Going the other direction (`pngcrush` first,
`pngquant` second) is also fine; the operations commute. JPEG /
WebP would be smaller again but lose the alpha channel; that
trade-off is what makes `pngquant` the only tool in this slot.

## Why use it

1. **Quality band exit codes are CI-friendly.** `--quality=65-80`
   means "if you can't make a palette that scores ≥65, bail
   without overwriting and exit 99". You wire that into a
   `Makefile` / CI step and the build refuses to publish a PNG
   that the quantiser itself thinks looks bad — no human in the
   loop, no eyeballing every diff.
2. **Alpha channel is first-class.** Quantisation respects
   premultiplied alpha and gamma, so semi-transparent edges of
   icons / UI screenshots / drop-shadows survive the palette
   reduction without the grey-fringing artefacts naïve
   quantisers produce. This is the reason it became the standard
   tool for design-system asset pipelines (vs. ImageMagick's
   `-colors 256` which is gamma-naïve).
3. **Composes cleanly with [`oxipng`](../oxipng/) /
   [`svgo`](../svgo/) / [`gifski`](../gifski/) /
   [`gifsicle`](../gifsicle/) into one Make pipeline.** Each
   tool owns one format and one operation; `pngquant` handles
   the lossy palette step for PNG, then `oxipng -o max` does
   the lossless re-encode pass for the last 5–10%, and the
   other formats route through their respective tools. There
   is no overlap and no policy conflict.

For an LLM-CLI workflow, `pngquant` exposes deterministic exit
codes (`0` ok, `99` quality gate failed, `98` cannot read input)
and structured stderr — an agent can decide whether to re-run with
a different `--quality` band, skip the file, or fall back to
WebP without scraping any text.

## Vs Already Cataloged

- **Vs [`oxipng`](../oxipng/):** `oxipng` is *lossless* — it
  re-encodes the same pixels with a better deflate / zopfli
  pass, typically saving 5–15%. `pngquant` is *lossy* — it
  changes the pixels (24-bit → palette+alpha) and saves
  60–80%. They are complementary: `pngquant` first for the big
  win, `oxipng -o max` second for the long tail. Running only
  `oxipng` on a 1.4 MiB photo PNG gets you ~1.2 MiB; running
  `pngquant` first and then `oxipng` gets you ~350 KiB on the
  same input.
- **Vs [`svgo`](../svgo/):** different format. `svgo` optimises
  vector SVGs (XML AST transforms); `pngquant` optimises raster
  PNGs (palette quantisation). A typical static-site build runs
  both: `svgo` over `src/icons/`, `pngquant + oxipng` over
  `src/img/`.
- **Vs [`gifski`](../gifski/) / [`gifsicle`](../gifsicle/):**
  different format (animated GIF). `gifski` actually shares the
  same `libimagequant` quantiser internally — the perceptual
  median-cut algorithm is the same; just adapted to per-frame
  palettes for GIF.
- **Vs `cwebp` / re-encoding to WebP / AVIF:** WebP / AVIF are
  smaller again on natural images, but a meaningful slice of
  consumers (older email clients, some PDF renderers, some
  CI-screenshot diff tools, some Apple ecosystem corners pre-
  iOS 14) still struggle with them. `pngquant` ships PNG-8,
  which everything everywhere supports — pick it when "must
  render in every browser ever made + must keep the alpha
  channel" is a hard constraint and "5% smaller than WebP at
  the same quality" is not.

## Caveats

- **Lossy.** A 24-bit photographic gradient compressed to a
  256-colour palette will show banding under heavy zoom no
  matter how clever the dithering. UI screenshots, icons,
  logos, illustrations — the long tail of "PNG of stuff that
  has solid colours and edges" — survive perfectly; high-
  frequency natural photography is where you should reach for
  WebP / AVIF / JPEG-XL instead.
- **GPL-3.0 plus a commercial license.** Running `pngquant` as
  a CLI in a CI build is fine (the asset is not a derived
  work). Linking `libimagequant` into a closed-source app is
  the case the commercial license exists for; do not statically
  link without checking. The CLI binary itself in package
  managers (Homebrew, apt, dnf) is GPL-3.0.
- **`--quality` is not 1:1 with JPEG quality.** The scale is
  similar in feel (0 = unusable, 100 = identical) but the
  numbers are not interchangeable; `--quality=70-85` is a good
  default starting band for most content. Test before pinning.
- **Default suffix is `-fs8.png` / `-or8.png`.** First-time
  users often don't notice the output went somewhere new and
  re-run `pngquant` on the *output*, getting a worse result.
  Use `--ext=.png --force` (or `-o explicit-name.png`) when
  scripting; never trust the default suffix in a `Makefile`.
- **Slow at `--speed=1`.** The default `--speed=4` is the right
  trade-off for batch builds; `--speed=1` is 5–10× slower for
  ~1–2% better quality and is worth it only for hand-pick
  hero assets, not for a 5000-file site asset pass.
- **Not a metadata tool.** `pngquant` strips PNG ancillary
  chunks behind `--strip`; it does not parse / rewrite EXIF /
  XMP / ICC profile content the way `exiftool` would. If you
  need fine-grained metadata control, do that pass before /
  after `pngquant`.
