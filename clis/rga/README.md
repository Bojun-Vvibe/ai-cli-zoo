# rga (ripgrep-all)

- **Repo:** https://github.com/phiresky/ripgrep-all
- **Version:** v0.10.10 (latest stable, 2025)
- **License:** AGPL-3.0 ([LICENSE.md](https://github.com/phiresky/ripgrep-all/blob/master/LICENSE.md))
- **Language:** Rust
- **Install:** `brew install rga` · `cargo install ripgrep_all` · `pacman -S ripgrep-all` · prebuilt binaries on the GitHub release page · binary name is `rga` (plus `rga-fzf` helper)

## What it does

`rga` is `ripgrep` with **adapters**: it transparently extracts text
from non-text files at search time and feeds the result through
`rg`, so a single `rga PATTERN .` searches inside PDFs, Office
documents (`.docx`, `.odt`, `.pptx`, `.xlsx`), EPUBs, ZIP / tar /
7z archives (recursively), SQLite databases, JPEG / PNG EXIF
metadata, audio / video container metadata (via `ffprobe`), and
optionally OCR'd images (via `tesseract`). Adapter output is cached
on disk (default: `~/.cache/ripgrep-all`) keyed by file content
hash, so the second search across the same corpus is `rg`-fast even
though the first run paid for PDF text extraction. All `rg` flags
(`-i`, `-C 2`, `--type`, `-l`, `--json`) work unchanged because
`rga` ultimately invokes `rg` underneath.

## When to pick it / when not to

Reach for `rga` when you have a directory of mixed-format reference
material — a research corpus, a downloads folder, an extracted
backup, a knowledge-base export — and you need "grep across all of
it including the PDFs and the zip files". It shines when you don't
know where the answer lives (which PDF page, which sheet of which
xlsx, which file inside which nested zip) and you want one ranked
list of hits with surrounding context.

Skip `rga` for plain source code trees — vanilla
[`ripgrep`](../ripgrep/) is faster and has nothing to extract.
Skip it for **semantic** search ("find passages about authentication
flow") — that is what [`seagoat`](../seagoat/), embedding indexes,
and RAG stacks are for; `rga` does literal / regex matching only.
Skip it on machines where you cannot install `pandoc`, `poppler`
(`pdftotext`), `ffmpeg`/`ffprobe`, and optionally `tesseract` —
those are the adapter binaries `rga` shells out to. Be aware of the
**AGPL-3.0** licence if you plan to embed `rga` itself in a
distributed product; using it as an end-user tool over your own
content is unaffected.

## Why it matters in an AI-native workflow

Agent context windows are finite, so before you can RAG over a
corpus you have to *find the right files*. `rga` is the lexical
pre-filter: an agent can run
`rga --json --max-count 5 'invoice number' ./drive-export` and get
back JSON-shaped hits (file path, page / sheet / row, surrounding
text) across PDFs, spreadsheets, and zipped archives in a single
call, then pass only the matched files to a more expensive
extraction or embedding step. The on-disk cache means the agent's
second search across the same corpus does not repay PDF
extraction cost, which matters when iteratively narrowing a query.
It complements [`fd`](../fd/) (file-name search),
[`ripgrep`](../ripgrep/) (text-only grep), and
[`ast-grep`](../ast-grep/) (structural code grep) on the lexical
side; the semantic side belongs to [`seagoat`](../seagoat/) and
friends.

## Example invocations

```bash
# Search a downloads folder including PDFs, docx, zips
rga 'invoice number' ~/Downloads

# Case-insensitive, with 2 lines of context, only show file names
rga -i -C 2 -l 'authentication' ./reference-docs

# JSON output for downstream tooling / agents
rga --json 'API_KEY' ./extracted-backup

# List which adapters will be used (and which are missing)
rga --rga-list-adapters

# Disable a specific adapter (e.g. skip OCR'ing images, which is slow)
rga --rga-adapters=-tesseract 'serial number' ./scans

# Enable an adapter that is off by default (e.g. OCR)
rga --rga-adapters=+tesseract 'serial number' ./scans

# Do NOT recurse into archives (treat .zip / .tar as opaque)
rga --rga-no-cache --rga-adapters=-zip,-tar 'pattern' .

# Pair with fzf for live preview across mixed-format content
rga --files-with-matches 'pattern' . | fzf --preview 'rga --pretty --context 5 {q} {}'
```

## Alternatives in this catalog

- [`ripgrep`](../ripgrep/) — the underlying engine; use it directly
  for source code trees where every file is already plain text.
- [`ast-grep`](../ast-grep/) — structural search by AST node, not
  by regex; for code refactors, not for cross-format text search.
- [`grep-ast`](../grep-ast/) — context-aware grep for source files
  (returns the enclosing function), complementary to `rga`'s
  cross-format reach.
- [`seagoat`](../seagoat/) — semantic / embedding-based code search
  when literal matches are not enough.
- [`fd`](../fd/) — file-name search; pair with `rga` to first
  narrow by path, then grep inside.
