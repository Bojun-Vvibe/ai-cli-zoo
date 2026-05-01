# pdfcpu

## What it does
A **pure-Go PDF processor** — single static binary plus library — that
parses, validates, and rewrites PDF files without shelling out to
Ghostscript / Poppler / Java. The CLI surface is verb-shaped:
`pdfcpu validate`, `pdfcpu optimize` (linearise + dedupe streams +
strip orphans, often 30–60 % smaller), `pdfcpu split` / `merge` /
`extract` (pages, images, fonts, metadata, attachments), `pdfcpu
encrypt` / `decrypt` / `permissions` (RC4-128 / AES-128 / AES-256),
`pdfcpu watermark` (text / image / PDF, per-page), `pdfcpu stamp`,
`pdfcpu rotate`, `pdfcpu nup` (n-up impose), `pdfcpu booklet`,
`pdfcpu form` (read / fill / export AcroForm + XFA values as JSON),
`pdfcpu portfolio`, `pdfcpu trim` / `collect` / `crop`. Conformance
target is the full PDF 1.7 / ISO 32000-1 spec plus chunks of PDF 2.0
read support; the validator is the reference implementation a number
of Go-based document pipelines (Caddy file server, static-site
generators, e-invoicing tooling) embed for "is this PDF actually a
PDF" gating.

## Why it's interesting
Different shape from Ghostscript (battle-tested but a 50+ MB C
toolchain with a dual GPL/AGPL/commercial licence and a long history
of PostScript-interpreter CVEs), from Poppler / `pdftk` (C++ wrapper
ergonomics, separate binary per verb, JNI for `pdftk`), from
`qpdf` (excellent for structural surgery + linearisation but no
watermark / nup / form-fill), from `mutool` (MuPDF — fast renderer
+ converter, less batch-rewrite focus, AGPL), and from cloud APIs
like Adobe PDF Services (network round-trip + per-call cost). pdfcpu
is the *one Apache-2.0 Go binary, no native deps, validator + writer
+ form filler in one process* shape: pick it specifically for CI
pipelines, container images, or scratch-from-distroless services that
need PDF transforms without dragging libgs / libpoppler in. Do **not**
pick it for high-fidelity rasterisation (`mutool` / Ghostscript
remain better renderers), for OCR (it doesn't do OCR — pair with
`tesseract` / [`marker`](../marker/)), or for content-aware editing
of glyph runs (the spec is too gnarly for any tool to do this
cleanly; export to source format and re-typeset with
[`typst`](../typst/) or LaTeX instead).

## Niche category
PDF toolkit — pure-Go single-binary validator + rewriter + form
processor that ships into containers and CI without native PDF
runtime dependencies.

## Repo
https://github.com/pdfcpu/pdfcpu

## Version pinned
`v0.11.0` (latest tagged release as of 2026-05-01)

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE.txt`

## Install
```sh
# Homebrew (macOS / Linux)
brew install pdfcpu

# Go install (any platform with a Go toolchain)
go install github.com/pdfcpu/pdfcpu/cmd/pdfcpu@v0.11.0

# Pre-built binaries for darwin / linux / windows / freebsd
# https://github.com/pdfcpu/pdfcpu/releases/tag/v0.11.0

# Container
docker run --rm -v "$PWD":/work pdfcpu/pdfcpu:0.11.0 validate /work/in.pdf
```

## Usage examples
```sh
# CI gate: fail the build if any PDF in dist/ is malformed
pdfcpu validate -mode strict dist/*.pdf

# Shrink a scan-heavy PDF (linearise + dedupe streams + drop unused objects)
pdfcpu optimize big.pdf small.pdf

# Apply a "DRAFT" diagonal watermark to every page
pdfcpu watermark add -mode text "DRAFT" \
  "rot:45, op:0.3, scale:0.9, font:Helvetica-Bold" \
  in.pdf out.pdf

# Split a quarterly report into one PDF per chapter on bookmark boundaries
pdfcpu split -mode bookmark report.pdf out/

# Read every AcroForm field value as JSON (pipe to jq for filtering)
pdfcpu form export filled.pdf | jq '.forms[0].fields[] | {name, value}'

# Merge invoices into one stitched PDF, preserving bookmarks
pdfcpu merge -mode create combined.pdf invoices/*.pdf
```

## Date added
2026-05-01
