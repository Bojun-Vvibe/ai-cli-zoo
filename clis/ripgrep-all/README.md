# ripgrep-all

> **`rga` — ripgrep for the rest of your filesystem**: a Rust
> wrapper around [`ripgrep`](https://github.com/BurntSushi/ripgrep)
> that transparently extracts text from PDFs, EPUB / MOBI e-books,
> Office documents (`.docx` / `.odt` / `.pptx` / `.xlsx`), zip /
> tar / gz / xz / 7z archives, sqlite databases, JPEG / PNG / HEIC
> images (via tesseract OCR), audio / video metadata (via
> ffprobe), and HTML — then runs the same `rg <pattern>` you
> already know across the extracted text, with results that point
> back at the *original* binary file. Pinned to **v0.10.10**
> (commit `e8cd5552379d60b12a17997177b7a8d34eedcdc4`,
> [LICENSE.md](https://github.com/phiresky/ripgrep-all/blob/v0.10.10/LICENSE.md),
> AGPL-3.0).

Source: <https://github.com/phiresky/ripgrep-all>

## TL;DR

`rga` is what you reach for when "I know the phrase is in *one*
of the 400 PDFs in `~/Documents/papers/`" but `rg` returns zero
hits because PDFs are binary blobs. `rga` shells out to the right
extractor per file type (`pdftotext`, `pandoc`, `unzip`,
`ffprobe`, `tesseract`), caches the extracted text on disk
keyed by content hash, and pipes the result through `rg` with
the original filename preserved — so `rga "fourier transform"
~/papers/` returns `papers/2019-vaswani.pdf:Page 4: …
fourier transform of the position embeddings …` in one
command, then the next run is instant because the extraction is
cached.

## Install

```bash
# Homebrew (macOS / Linux)
brew install rga

# Cargo (with the right adapter binaries on PATH)
cargo install --locked ripgrep_all

# Linux package managers
# Arch:    pacman -S ripgrep-all
# Nix:     nix-env -iA nixpkgs.ripgrep-all
# Alpine:  apk add ripgrep-all

# from a release tarball (pre-built x86_64 / aarch64)
curl -Lo rga.tar.gz "https://github.com/phiresky/ripgrep-all/releases/download/v0.10.10/ripgrep_all-v0.10.10-aarch64-apple-darwin.tar.gz"
tar xf rga.tar.gz
sudo install ripgrep_all-v0.10.10-aarch64-apple-darwin/rga /usr/local/bin/

# verify
rga --version    # rga 0.10.10
```

`rga` requires `rg` plus the per-format adapter binaries
(`pdftotext` / `pandoc` / `tesseract` / `ffprobe` / `7z` /
`xlsx2csv`) on `$PATH`. Brew installs them as recommended deps;
on Linux install the `poppler-utils` / `pandoc` / `tesseract` /
`ffmpeg` / `p7zip` packages.

## License

AGPL-3.0 — see
[LICENSE.md](https://github.com/phiresky/ripgrep-all/blob/v0.10.10/LICENSE.md).
The AGPL clause is the trade-off; if you ship `rga` *as a
network service* you must offer source. For interactive shell /
laptop use the obligation does not bite.

## One Concrete Example

```bash
# 1. search every supported file format under cwd
rga "transformer architecture" .

# 2. limit to PDFs only (skip everything else for speed)
rga --rga-adapters=+pdfpages "attention is all you need" ~/papers/

# 3. list which adapters are active and which extensions they cover
rga --rga-list-adapters

# 4. stream raw extracted text instead of grep matches
rga --rga-no-cache --rga-adapters=+postprocpagebreaks "" some.pdf

# 5. plain ripgrep flags pass through untouched
rga -i -C 2 --type-add 'doc:*.{pdf,docx,epub}' -tdoc "fourier" .
```

## Why This Over `rg` Alone

| ask                                          | answer       |
| -------------------------------------------- | ------------ |
| grep recursively in source code              | `rg`         |
| grep across 400 PDFs                         | `rga`        |
| grep inside a zip without unpacking it first | `rga`        |
| OCR scanned-image PDFs and grep the text     | `rga`        |
| grep `.docx` / `.epub` / `.pptx`             | `rga`        |
| grep a multi-GB sqlite dump                  | `rga`        |
| do all of the above with `rg`'s perf budget  | `rga` (cached) |

`rga` does not replace `rg` — it composes with it. Every flag
you know (`-C`, `-i`, `--type`, `-l`, JSON output) works
unchanged.

## Caveats

- First run on a large corpus is slow because every file gets
  extracted; subsequent runs hit the on-disk cache (default
  `~/.cache/rga`, override with `--rga-cache-dir`).
- OCR (`tesseract`) is the slowest adapter by an order of
  magnitude. Disable per-run with `--rga-adapters=-tesseract`
  if you only care about text-layer PDFs.
- AGPL-3.0 means embedding `rga` inside a SaaS without exposing
  modified source breaches the license. For laptops / CI it's a
  non-issue.
- The adapter binaries (`pdftotext` / `pandoc` / etc.) are
  external; if `pdftotext` segfaults on a malformed PDF, `rga`
  surfaces the error and skips that file — it does not retry.
