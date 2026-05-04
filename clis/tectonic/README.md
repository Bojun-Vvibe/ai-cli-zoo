# tectonic

> **Modernized, self-contained TeX/LaTeX engine** — fork of XeTeX
> that auto-fetches missing packages from a TeX Live web bundle,
> runs the document to convergence in one command, and ships as a
> single Rust binary. Pinned to **v0.15.0** (released 2024-03-30,
> [LICENSE](https://github.com/tectonic-typesetting/tectonic/blob/master/LICENSE),
> MIT).

Source: <https://github.com/tectonic-typesetting/tectonic>

## TL;DR

`tectonic` replaces the "install a 5 GB TeX Live distribution,
run `pdflatex` three times, then `bibtex`, then `pdflatex` twice
more" ritual with `tectonic doc.tex`. On first run it downloads
only the `.sty` / `.cls` / font files your document actually
references from a curated web bundle, caches them locally
(`~/Library/Caches/Tectonic` on macOS), and reruns the engine
internally until labels and references converge — so the first
build of a fresh document on a fresh machine is one command and
typically under 30 seconds, and subsequent builds are offline.
The `tectonic -X` "V2" interface adds a workspace model
(`Tectonic.toml`) so you can describe multi-document books and
build them reproducibly. Output is PDF; the engine is XeTeX-based
so Unicode, OpenType, and system fonts work without configuration.

## Why pick it over alternatives

Pick `tectonic` when you want LaTeX output but refuse to manage a
full TeX Live install, or when you need reproducible builds in CI
(Docker images go from ~4 GB to ~80 MB). Compared to full
**TeX Live + latexmk**: TeX Live still wins for exotic packages,
TikZ-heavy academic papers with bleeding-edge dependencies, and
shops that already have a working install — tectonic's bundle is
curated and lags upstream by months. Compared to
[`typst`](../typst/) and [`tinymist`](../tinymist/): typst is a
*new* typesetting language with a faster compiler and saner error
messages, but it is not LaTeX — your existing `.tex` files,
journal templates, and `\usepackage{...}` lines do not port.
Compared to **pandoc → LaTeX → pdflatex**: pandoc still drives
the pipeline; tectonic just replaces the `pdflatex` leg.
Skip tectonic if your workflow depends on packages outside the
TeX Live bundle (rare custom `.sty` files), if you need DVI
output, or if you are already happy with your local TeX Live.

## Install

```bash
# macOS / Linux (single binary)
brew install tectonic

# Cargo
cargo install tectonic

# one-shot installer
curl --proto '=https' --tlsv1.2 -fsSL https://drop-sh.fullyjustified.net | sh

# verify
tectonic --version    # 0.15.0
```

Quick start:

```bash
# compile a single .tex file (auto-fetches packages on first run)
tectonic paper.tex
# → paper.pdf, no extra runs needed

# V2 workspace mode
tectonic -X new mybook
cd mybook
tectonic -X build

# keep intermediates for debugging
tectonic --keep-intermediates --keep-logs paper.tex
```
