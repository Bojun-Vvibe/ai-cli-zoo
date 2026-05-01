# pandoc

- **Repo:** https://github.com/jgm/pandoc
- **Version:** v3.9.0.2 (released 2026-03-19, latest stable as of cataloging)
- **License:** GPL-2.0-or-later ([COPYING.md](https://github.com/jgm/pandoc/blob/main/COPYING.md), with the file-format-specific notices in [COPYRIGHT](https://github.com/jgm/pandoc/blob/main/COPYRIGHT))
- **Language:** Haskell
- **Install:** `brew install pandoc` · `apt install pandoc` · `pacman -S pandoc` · prebuilt `.deb` / `.pkg` / `.msi` / static tarballs on the GitHub release page · binary name is `pandoc`

## What it does

`pandoc` is the universal document converter: a Haskell program
built around an internal AST that has **readers** for ~40
input formats (Markdown in several dialects, reStructuredText,
AsciiDoc, Org, LaTeX, HTML, DocBook, Textile, MediaWiki, Jira,
Roff, Typst, Docx, ODT, EPUB, FB2, OPML, CSV, JATS, JSON, …)
and **writers** for an even larger set of output formats
(everything above plus PDF via several engines, PowerPoint,
ICML, ConTeXt, beamer slides, reveal.js / Slidy / Slideous /
DZSlides / S5 HTML decks, man pages, info, custom Lua writers).
Conversion is reader → AST → optional Lua filter → writer, so
any input can target any output, and the AST stage is where
custom filters insert citations (built-in `--citeproc` with CSL
styles), rewrite headings, transform code blocks, or splice in
generated content. Templates per output format control the
final shape (LaTeX preamble, HTML head, EPUB metadata) without
touching the document body. Math is round-tripped through
MathML / MathJax / KaTeX / LaTeX as the target requires.

## When to pick it / when not to

Reach for `pandoc` when you need to **author once and emit many
formats**: a single Markdown source becomes a PDF, an EPUB, a
slide deck, a Word `.docx` for review, and a static HTML page.
It is the right tool when the conversion has to be scripted and
reproducible (Makefiles, CI-built docs sites, academic theses
with bibliography), when the input is a file format no other
converter handles cleanly (Org → Docx, JATS → Markdown,
DocBook → EPUB), and when filters need to programmatically
transform the document (Lua filters operate on the parsed AST,
which is far more reliable than regex-on-source).

Skip it for a static-site generator workflow where the site
generator already does Markdown → HTML well (use Hugo / Zola /
Astro and let them call pandoc only for the formats they don't
handle). Skip it for "make this PDF look exactly like the Word
template" — pandoc's PDF output is excellent for content-driven
documents but will fight you on pixel-perfect designer layouts;
use the source application for those. Note: the binary is
~30–40 MB and Haskell-built, so cold start is slower than
single-purpose Markdown-to-HTML tools — fine for batch jobs,
noticeable in tight per-keystroke loops.

## Why it matters in an AI-native workflow

LLM output is overwhelmingly Markdown, but downstream consumers
want PDF reports, `.docx` review drafts, EPUB study guides, and
slide decks. `pandoc` is the conversion layer that turns one
LLM-produced Markdown stream into all of those without a second
generation pass. Lua filters let an agent insert generated
citations, transform code blocks into syntax-highlighted figures,
or splice machine-generated diagrams (Mermaid, D2) at the AST
stage rather than asking the model to emit format-specific
markup. The `--citeproc` engine with a CSL bibliography turns
"please format these references in APA" from a hallucination
risk into a deterministic post-process.

## Example invocations

```bash
# Markdown to a self-contained HTML file with embedded CSS
pandoc input.md -s --embed-resources -o output.html

# Markdown to PDF via the default LaTeX engine (needs TeX Live)
pandoc input.md -o output.pdf

# Markdown to PDF without a TeX install, via the typst engine
pandoc input.md --pdf-engine=typst -o output.pdf

# Markdown to .docx using a corporate reference template
pandoc input.md --reference-doc=template.docx -o output.docx

# Academic paper with citations and a CSL style
pandoc paper.md --citeproc --bibliography=refs.bib \
  --csl=ieee.csl -o paper.pdf

# Markdown to a reveal.js slide deck
pandoc slides.md -t revealjs -s -o slides.html

# Read DocBook, write Markdown — the catch-all migration command
pandoc -f docbook -t gfm in.xml -o out.md

# Apply a Lua filter that, e.g., demotes every heading by one level
pandoc input.md --lua-filter=demote.lua -o output.md
```

## Alternatives in this catalog

- [`mdcat`](../mdcat/) — terminal-only Markdown renderer; pick it
  to *read* Markdown in a shell, pick pandoc to *convert* it.
- [`glow`](../glow/) — TUI Markdown reader, same niche as mdcat.
- [`d2`](../d2/) — diagram-as-code; pairs with pandoc as a Lua
  filter to render `d2` code blocks into embedded SVGs.
- [`asciinema`](../asciinema/) — terminal session recording;
  unrelated, but pandoc can convert the JSON cast notes around it.
