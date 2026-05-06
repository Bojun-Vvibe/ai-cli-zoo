# sile

> **sile** — a modern typesetter in the spirit of TeX, written in Lua,
> aimed at producing book-quality PDF from structured input (its own
> SIL markup or XML / Markdown via packages). Pinned to **v0.15.13**,
> MIT — license file:
> [LICENSE.md](https://github.com/sile-typesetter/sile/blob/master/LICENSE.md).

Source: <https://github.com/sile-typesetter/sile>

## TL;DR

`sile input.sil -o output.pdf` runs the SILE typesetter and produces
print-ready PDF using HarfBuzz for shaping, ICU for text breaking,
and Lua as the extension language — so the page-layout, line-break,
hyphenation, and font-feature decisions are TeX-class but the
*scripting surface* is plain Lua, not the TeX macro language. Where
LaTeX answers "I want my paper to look like every other LaTeX paper",
SILE answers "I want a custom book layout (poetry collection,
bilingual prayer book, music theory text, RTL/LTR mixed Hebrew or
Arabic, complex Indic shaping) and I want to write the layout in a
language a working programmer already knows".

The model is a `\command{...}` markup tree (SIL syntax) or its XML
equivalent that gets parsed into a node graph and run through a
class (book, plain, jbook, …) plus a stack of packages
(`grid`, `verse`, `cropmarks`, `pdf`, `bibtex`, `hanmenkyoshi`, …)
that hook into the typesetter. Pandoc can emit SIL, so a Markdown
manuscript can be typeset via `pandoc -t sile manuscript.md | sile`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install sile

# Nix
nix-env -iA nixpkgs.sile

# Pre-built binaries
# https://github.com/sile-typesetter/sile/releases/tag/v0.15.13
```

Hard runtime dependencies: HarfBuzz, ICU, FreeType, fontconfig (or
the bundled font on macOS), and a Lua 5.4 / LuaJIT interpreter.

## Example commands

```bash
# Typeset an SIL document to PDF
sile book.sil -o book.pdf

# Use a different class
sile -c jbook chapter1.sil

# Pandoc → SILE pipeline for Markdown manuscripts
pandoc -t sile manuscript.md | sile - -o manuscript.pdf

# List available classes / packages
sile --help
```

A minimal SIL document:

```sil
\begin{document}
\font[family=EB Garamond,size=12pt]
\chapter{The opening}
First paragraph with proper line breaking, hyphenation, and
ligatures via HarfBuzz.
\end{document}
```

## Niche it occupies

**Programmable, font-aware book typesetter** — closest neighbour
in the catalog is [`tectonic`](../tectonic/) (a Rust-rewritten
LaTeX engine with self-fetching package cache), and
[`tinymist`](../tinymist/) / [`typst`](../typst/) (a modern
typesetting system in Rust with its own markup language).

- Pick **sile** when the layout is *non-standard* (verse, parallel
  text, complex script shaping, custom page furniture) and you want
  to write the layout decisions in Lua, not in TeX's macro language
  — and when LaTeX's class library doesn't already contain what you
  want.
- Pick [`tectonic`](../tectonic/) when the input you have is
  LaTeX (existing journal templates, thesis classes, BibTeX-shaped
  references) and "build a PDF from this LaTeX without me installing
  TeX Live" is the verb.
- Pick [`typst`](../typst/) / [`tinymist`](../tinymist/) when you
  want a modern, fast incremental typesetter with a designed-from-scratch
  markup + scripting language and the LSP / preview surface that
  comes with it; pick SILE when you want Lua as the extension
  language and an XML-shaped input mode that fits cleanly into an
  XSLT / Pandoc pipeline.
- Orthogonal to [`pandoc`](../pandoc/) — SILE is one of pandoc's
  output backends, so SILE owns the *typesetting* and pandoc owns
  the *source-format conversion*.

Pairs with [`presenterm`](../presenterm/) (slides from Markdown;
sile is for the *book*-shaped artifact), and with
[`ghostscript`](https://www.ghostscript.com/) / [`qpdf`](../qpdf/)
for post-processing the PDF SILE emits.

## Caveats

- The package ecosystem is *much* smaller than LaTeX's CTAN — for
  a journal template that already exists in LaTeX, tectonic is the
  faster path.
- Lua 5.4 + LuaJIT compatibility means some packages run only on
  one of the two; check the package's manifest before pinning.
- Cold-start cost is non-trivial on a small document compared to
  Typst's incremental compiler — SILE shines on book-shaped jobs,
  not on rapid edit-preview loops.

## Citation

- Repo: <https://github.com/sile-typesetter/sile>
- Latest release: **v0.15.13**
- License: **MIT**
- License file: [LICENSE.md](https://github.com/sile-typesetter/sile/blob/master/LICENSE.md)
