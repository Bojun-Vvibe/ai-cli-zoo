# qpdf

> **Content-preserving PDF transformation toolkit** — a C++ CLI
> (and library) that reads PDF, manipulates structure
> (linearization, encryption, decryption, page splitting/merging,
> object stream rewriting, JSON inspection), and writes PDF —
> *without* re-rendering content or rasterizing pages, so output
> is byte-faithful where it can be and structurally clean where it
> cannot. Pinned to **v12.3.2** (released 2025-08-23,
> [LICENSE.txt](https://github.com/qpdf/qpdf/blob/main/LICENSE.txt),
> Apache-2.0).

Source: <https://github.com/qpdf/qpdf>

## TL;DR

`qpdf` is the answer when you want to *do something to a PDF*
without going through Ghostscript (which re-rasterizes and
silently downgrades quality) or a full PDF library binding
(pikepdf, PyPDF2, MuPDF). It linearizes (a.k.a. "fast web view"),
encrypts/decrypts, splits/merges/reorders pages, attaches files,
removes restrictions, fixes broken cross-reference tables on
truncated downloads, and emits JSON descriptions of internal
PDF structure for diff / audit / debug. It does *not* render,
does *not* re-encode images, does *not* parse text content
streams beyond what's needed for structure — which is exactly
why archivists, legal-discovery teams, and prepress shops trust
it where they distrust Ghostscript.

## Install

```bash
# macOS / Linux
brew install qpdf

# Debian / Ubuntu
sudo apt install qpdf

# Fedora
sudo dnf install qpdf

# Arch
sudo pacman -S qpdf

# Windows
# winget install qpdf.qpdf
# (or download MSI from https://github.com/qpdf/qpdf/releases)

# verify
qpdf --version    # qpdf version 12.3.2
```

## Examples

```bash
# remove a known owner password (decrypt) — for files YOU own
qpdf --password=secret --decrypt locked.pdf unlocked.pdf

# add 256-bit AES encryption with separate user + owner passwords
qpdf --encrypt user_pw owner_pw 256 -- input.pdf encrypted.pdf

# linearize for fast web view (so the first page renders before
# the whole file finishes downloading)
qpdf --linearize input.pdf web-optimized.pdf

# split a PDF into one file per page
qpdf input.pdf --split-pages=1 page-%d.pdf

# merge / reorder: pages 1-5 of a.pdf, then all of b.pdf, then
# pages 10-12 of a.pdf reversed
qpdf --empty --pages a.pdf 1-5 b.pdf 1-z a.pdf 12-10 -- merged.pdf

# repair a truncated / damaged PDF by rebuilding xref tables
qpdf --check broken.pdf
qpdf broken.pdf repaired.pdf

# dump the document structure as JSON for scripting / diff
qpdf --json input.pdf | jq '.objects | keys | length'

# strip all attachments and JavaScript actions
qpdf --remove-attachments --no-original-object-ids in.pdf clean.pdf
```

## Use when

- You need lossless PDF surgery: split, merge, reorder, encrypt,
  decrypt, linearize, attach, repair — without re-rendering and
  without dragging in Ghostscript's licensing or quality-loss
  surprises.
- You are building a document pipeline (legal discovery, tax
  filing, archival ingest) and need deterministic, scriptable
  PDF transformations that produce byte-stable output.
- A PDF arrived corrupted (truncated download, killed writer)
  and you need to recover what's there — `qpdf` rebuilds the
  cross-reference table from a stream scan, often where
  Acrobat refuses to open the file.
- You need to inspect PDF internals (object streams, page tree,
  encryption dict) for security audit or forensic work — the
  `--json` and `--qdf` modes expose the structure cleanly.

Skip `qpdf` for content-level work: extracting text → use
`pdftotext` (poppler) or `mutool draw` (mupdf); rendering to
images → use `pdftoppm` or `mutool convert`; OCR → `ocrmypdf`
(which itself uses `qpdf` internally for the structural
rewrites). For high-level Python scripting,
[`pikepdf`](https://github.com/pikepdf/pikepdf) wraps `qpdf` as
a library — same engine, idiomatic API.
