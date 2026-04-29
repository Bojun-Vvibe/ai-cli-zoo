# oxipng

> **Multithreaded lossless PNG optimizer in Rust** — a single
> binary that re-encodes PNG files with smaller IDAT chunks
> (better filter selection, better deflate, optional zopfli) and
> optionally strips non-essential metadata (gAMA, cHRM, EXIF,
> textual chunks). A spiritual successor / cleanroom rewrite of
> `optipng` that parallelises filter trial across cores. Pinned
> to **v9.1.5** (SPDX: `MIT`,
> [LICENSE](https://github.com/shssoichiro/oxipng/blob/master/LICENSE)).

Source: <https://github.com/shssoichiro/oxipng>

## TL;DR

`oxipng` is what you reach for when you have a directory full of
PNGs (screenshots, exported chart frames, sprite atlases, web
assets) and you want them 5–40 % smaller without any visual
difference. It tries multiple deflate filter strategies in
parallel, keeps the best one, and optionally re-runs the result
through `zopfli` for a final few percent at the cost of CPU. No
quality knob — output is byte-for-byte equivalent at the pixel
level.

## Install

```bash
# Homebrew (macOS / Linux)
brew install oxipng

# Cargo
cargo install --locked oxipng

# Linux package managers
# Arch:    pacman -S oxipng
# Debian/Ubuntu (24.04+): apt install oxipng
# Fedora:  dnf install oxipng
# Nix:     nix-env -iA nixpkgs.oxipng
# Alpine:  apk add oxipng

# Pre-built release binary
curl -Lo oxipng.tar.gz "https://github.com/shssoichiro/oxipng/releases/download/v9.1.5/oxipng-9.1.5-aarch64-apple-darwin.tar.gz"
tar xf oxipng.tar.gz --strip-components=1
sudo install oxipng /usr/local/bin/

# verify
oxipng --version    # oxipng 9.1.5
```

## License

MIT — see
[LICENSE](https://github.com/shssoichiro/oxipng/blob/master/LICENSE).

## One Concrete Example

```bash
# 1. shrink a single PNG in place at default optimisation (-o 2)
oxipng image.png

# 2. strip ALL safe-to-remove metadata, max optimisation, recurse
oxipng -o max --strip safe -r ./assets/

# 3. extra slow zopfli pass (squeeze last ~2-5 %), keep originals
oxipng -Z --out optimised/ ./*.png

# 4. dry run — report savings without writing
oxipng --pretend -r ./screenshots/

# 5. parallel batch over a project, log only the wins
oxipng -o 4 -r ./public/ 2>&1 | grep -E "saving|reduction"

# 6. pre-commit hook style: only rewrite if smaller
oxipng --opt 3 --strip safe --preserve *.png
```

## Niche It Fills

**Lossless PNG re-encoding, parallel by default.** When you
control the source (screenshots, generated charts, design
exports) and want bytes back without changing pixels, `oxipng`
is the modern default. It is *not* a converter (use `cwebp` /
`avifenc` for that), *not* a lossy quantiser (use `pngquant`
for palette reduction), and *not* a viewer.

## Why use it

1. **Parallel filter trial.** Tries multiple PNG row-filter
   strategies across all cores, picks the smallest. `optipng`
   trials sequentially.
2. **Safe metadata stripping with named profiles.** `--strip
   safe` removes only chunks known not to affect rendering;
   `--strip all` removes everything non-critical. Beats hand-
   rolled `exiftool` pipelines.
3. **Zopfli mode.** `-Z` swaps in zopfli for the final deflate
   pass — extra 2–5 % at 10–100× the CPU. Worth it for
   ship-once assets (web hero images, app icons).
4. **`--pretend`.** Reports per-file savings without writing,
   so you can budget a CI job before letting it touch files.

## Vs Already Cataloged

- **Vs [`ffmpeg`](../ffmpeg/):** orthogonal — `ffmpeg` will
  re-encode PNGs but is overkill and not as small as `oxipng`
  for the lossless case.
- **Vs [`imagemagick`](../imagemagick/) / `magick`:** `magick`
  can write PNGs but does not exhaustively trial filter
  strategies; `oxipng` output is consistently smaller for the
  same pixels.

## Caveats

- **Lossless only.** If your PNG is a photograph with millions
  of colours, the win will be small (1–5 %). Convert to WebP /
  AVIF / JPEG-XL via `cwebp` / `avifenc` for real savings.
- **`--strip all` removes colour profile (iCCP) and gamma
  (gAMA).** Photo-accurate workflows want `--strip safe` (which
  keeps iCCP) or no stripping at all.
- **In-place by default.** Use `--out` or `--dir` to write
  elsewhere; use `--preserve` to keep file mtime.
- **`-Z` (zopfli) is slow.** Tens of seconds per medium PNG.
  Reserve for build artefacts, not interactive runs.
