# pdfgrep

> **A `grep` that reads PDF files — same regex syntax,
> same `-r` recursion, same colored output, but extracts
> text from each page first and reports matches with
> page-number context** (`--page-number`, `--page-range`,
> optional `--cache` for re-runs over the same corpus).
> Single C++ binary built on Poppler + PCRE2. Pinned to
> **v2.2.0**
> ([COPYING](https://gitlab.com/pdfgrep/pdfgrep/-/blob/master/COPYING),
> GPL-2.0-or-later).

Source: <https://gitlab.com/pdfgrep/pdfgrep>

## TL;DR

You have a folder of papers, datasheets, contracts, manuals,
or scanned-and-OCR'd PDFs and you want to find the page that
mentions `idle_timeout` or `clause 7.3` or `Theorem 4`.
`grep` does not see PDF; `pdftotext file.pdf - | grep` works
once but loses page numbers, bombs on encrypted files, and
gives you nothing for `-r ./papers/`. `pdfgrep` is the
straight answer: same flag set as GNU grep where it makes
sense (`-i`, `-n`, `-r`, `-l`, `-c`, `-C`, `-A`, `-B`, `-E`,
`-P`, `--include`, `--exclude`, `--color=auto`), with PDF-
specific extras (`--page-number`, `--page-range 5-20`,
`--password`, `--cache` to memoize text extraction across
runs). Output looks identical to `grep`, prefixed with the
page number, so it slots into every existing grep-aware
pipeline (`fzf`, editor jump-to-line wrappers, `xargs -0`).

## Install

```bash
# Homebrew (macOS / Linux)
brew install pdfgrep

# Debian / Ubuntu
sudo apt install pdfgrep

# Arch Linux
pacman -S pdfgrep                 # extra repo

# Fedora
sudo dnf install pdfgrep

# From source (autotools, needs poppler + pcre2)
git clone https://gitlab.com/pdfgrep/pdfgrep
cd pdfgrep
git checkout v2.2.0
./autogen.sh
./configure
make -j4
sudo make install

# verify
pdfgrep --version    # pdfgrep 2.2.0
```

The Homebrew / distro packages already pull Poppler and
PCRE2 transitively. From source you need
`libpoppler-cpp-dev` (Debian name) or `poppler` (Homebrew)
and `libpcre2-dev` available before `./configure`.

## Use it for

```bash
# Search one PDF
pdfgrep 'idle_timeout' router-manual.pdf

# Show page numbers (the whole point)
pdfgrep -n 'Theorem 4' paper.pdf
# paper.pdf:14: Theorem 4 (Convergence). Suppose ...

# Recursive across a paper library, just like grep -r
pdfgrep -rn 'attention is all you need' ~/papers/

# Restrict to filename glob (default is all *.pdf / *.PDF)
pdfgrep -rn --include='*-2025-*.pdf' 'methodology' ~/papers/

# Only list filenames that match (xargs-friendly)
pdfgrep -rl 'GDPR' ~/contracts/ | xargs -I{} echo "review: {}"

# Count matches per file
pdfgrep -rc 'TODO' ~/spec-drafts/

# Context lines, exactly like grep -A / -B / -C
pdfgrep -rn -C2 'kernel panic' ~/datasheets/

# Extended regex (-E) and Perl regex (-P, via PCRE2)
pdfgrep -rn -E 'CVE-202[0-9]-[0-9]{4,7}' ~/security-bulletins/
pdfgrep -rn -P '(?<=Section\s)\d+(\.\d+)+' ~/specs/

# Limit by page range — skip the 200-page appendix
pdfgrep --page-range 1-40 -n 'introduction' textbook.pdf

# Page label instead of page index (book numbering, "iv", "12-A")
pdfgrep --page-number=label -n 'preface' textbook.pdf

# Encrypted PDFs
pdfgrep --password=secret -n 'amount due' invoice.pdf

# Cache extracted text for fast re-runs over the same corpus
pdfgrep --cache -rn 'eigenvalue' ~/papers/        # first run: slow
pdfgrep --cache -rn 'gradient' ~/papers/          # later runs: fast

# Pipe into fzf for an interactive picker, then open at the page
pdfgrep -rn 'attention' ~/papers/ \
    | fzf \
    | awk -F: '{ system("zathura --page="$2" \""$1"\"") }'
```

The match-line format is identical to `grep`'s
(`<file>:<page>: <text>`), so anything that already
processes grep output (`xargs`, `awk -F:`, editor
quickfix lists, `fzf --preview`) handles `pdfgrep`'s
output unchanged. The `--cache` flag stores extracted text
under `~/.cache/pdfgrep/` keyed by file mtime + size, which
turns a 30-second recursive search over a 2 GB papers
folder into a sub-second one on the second run.

## Why include it in a CLI catalog

1. **It is the canonical "grep my PDF library" tool, with
   no real competitor in this niche.** [`ripgrep`](../ripgrep/)
   does not read PDF; [`ripgrep-all`](../ripgrep-all/)
   (rga) does, by shelling out to `pdftotext`, but loses
   page-number context. `pdfgrep` is the only widely
   packaged tool that produces grep-style output *with
   page numbers*, which is exactly what you need to jump
   straight to the page in your viewer.
2. **GNU-grep flag fidelity.** `-i`, `-n`, `-r`, `-l`,
   `-c`, `-A`, `-B`, `-C`, `-E`, `-P`, `--include`,
   `--exclude`, `--color=auto`, `-Z` (NUL output for
   xargs) — they all behave the way you expect. Muscle
   memory transfers cleanly; you do not have to learn a
   new flag dialect to search a folder of PDFs versus a
   folder of source files.
3. **`--cache` makes interactive workflows viable.** Text
   extraction is the slow step (Poppler parses the page
   stream, runs through fonts, decodes CIDs). With
   `--cache`, the second `pdfgrep` over the same papers
   directory is bounded by regex cost, not extraction
   cost — a 50× speed-up on a real research library.
   `fzf` + `pdfgrep --cache` becomes a pleasant
   interactive PDF search.

For an LLM-CLI workflow, `pdfgrep -rn -Z 'pattern'
~/papers/` produces NUL-separated `<file>:<page>:<text>`
records that an agent can split deterministically and feed
back into a "fetch page N of file F as text" step
(`pdftotext -f N -l N file.pdf -`) for a precise
context window — much cheaper than dumping every PDF
into the prompt.

## Vs Already Cataloged

- **Vs [`ripgrep`](../ripgrep/):** orthogonal —
  `ripgrep` is the reference recursive grep, but it
  treats PDFs as binary noise. Use `ripgrep` for source
  trees and plain text; use `pdfgrep` for the
  `~/papers/`, `~/datasheets/`, `~/contracts/` folders.
- **Vs [`ripgrep-all`](../ripgrep-all/) (rga):** closest
  peer — `rga` is a `ripgrep` adapter that handles many
  binary formats (PDF, DOCX, EPUB, ZIP, sqlite) by
  shelling out to extractors. It is broader but loses
  PDF page-number information (it gives byte offsets
  into the extracted text). Pick `rga` when "I want to
  grep across PDFs *and* DOCX *and* EPUB at once"; pick
  `pdfgrep` when the job is PDF-only and you need to jump
  to a page.
- **Vs [`pdftotext`](https://poppler.freedesktop.org/) +
  `grep`:** orthogonal — the manual two-step works for
  one file, but you lose page numbers, the recursion,
  the `-r --include` filtering, the encryption support,
  and the cache. `pdfgrep` is essentially the productized
  version of that pipeline, and it is built on the same
  Poppler library, so extraction quality is identical.
- **Vs full-text search engines (Recoll, Apache Tika,
  Meilisearch on extracted PDF text):** orthogonal —
  those build a persistent index and re-rank by score
  across many file types. `pdfgrep` is the
  zero-configuration "I just want to grep a folder of
  PDFs from the shell, today" tool. For a corpus that
  grows past ~10k PDFs and is queried by multiple
  people, an indexer wins; below that, `pdfgrep --cache`
  is enough.

## Caveats

- **Scanned-image PDFs are invisible without OCR first.**
  `pdfgrep` reads the *text layer* of a PDF. A PDF that
  is just a stack of page images (typical scan) has no
  text layer and matches nothing. Run `ocrmypdf in.pdf
  out.pdf` first, then `pdfgrep` over `out.pdf`.
- **GPL-2.0-or-later license.** Stricter than the
  MIT / Apache-2.0 norm in this catalog; for
  redistribution inside a proprietary product, read
  COPYING. Pure end-user CLI use is unaffected.
- **`--cache` is invalidated by mtime + size.** If a PDF
  is touched (e.g. metadata-only edit) the cache entry
  is rebuilt. If two different PDFs happen to have the
  same size and mtime path, the cache key is still
  unique because it includes the absolute path; but
  copying a corpus to a new machine resets every cache
  entry. Re-warming on first search is normal.
- **PCRE2 syntax under `-P` differs subtly from PCRE1.**
  Older `pdfgrep` releases linked PCRE1; v2.x links
  PCRE2. Most patterns transfer, but a handful of
  obscure constructs (`(?R0)`, certain
  callout syntaxes) behave differently. Test patterns
  before scripting them into a pipeline.
- **`--page-number=label` only works when the PDF
  declares page labels.** Many PDFs do not, in which
  case `--page-number=label` falls back silently to the
  numeric index. If labels matter (book scans with roman
  numeral front matter), inspect with `pdfinfo file.pdf`
  first to confirm `Page Labels: yes`.
- **Last release v2.2.0 (2024-12).** Maintenance cadence
  is slow but the codebase is mature; the project has
  been stable since the 2.x line and most distros ship
  it directly.
