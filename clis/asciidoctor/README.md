# asciidoctor

- **Upstream:** https://github.com/asciidoctor/asciidoctor
- **Version:** v2.0.26 (latest stable as of 2026-05-05)
- **License:** MIT — [LICENSE](https://github.com/asciidoctor/asciidoctor/blob/main/LICENSE)

## What it does

`asciidoctor` is a Ruby-implemented processor for the **AsciiDoc**
lightweight markup language — a richer cousin of Markdown designed
specifically for technical documentation, with first-class support for
nested admonitions, callout-annotated source listings, conditional
includes, multi-level numbered section refs, footnotes, bibliographies,
indices, mathematical equations (via STEM blocks), tables with cell
spans / cell formatting / CSV / DSV input, and an extensible attribute
substitution engine that lets a doc author parameterise the same source
across product editions. The CLI takes one or more `.adoc` files and
emits HTML5 by default (`asciidoctor doc.adoc -o doc.html`), with
`-b docbook` producing the DocBook XML that the broader documentation
toolchain (`pandoc`, `dblatex`, `xsltproc`) consumes for downstream
PDF / EPUB / man-page rendering, `-b manpage` producing a roff man page
directly, and the sibling `asciidoctor-pdf` gem producing PDF natively
without a LaTeX dependency. The processor is what GitHub, GitLab,
Sourcehut, the Linux kernel docs (since the 2018 switch off DocBook
sources), and the Asciidoctor-built sites of Spring, Gradle, Hibernate,
ArchWiki-adjacent projects, OpenJDK proposals, and the AsciiDoc
Language Specification itself all use to render `.adoc` source.

## Why it's interesting / niche

The existing document-conversion stack in the zoo (`pandoc`, `typst`,
plus the prose-tooling family) covers Markdown / reStructuredText /
LaTeX / Typst end-to-end, but the **AsciiDoc** input lane is empty —
and AsciiDoc is the dominant authoring format for the documentation
sites of large open-source Java / Spring / Gradle / Linux-kernel
ecosystems where Markdown's expressive ceiling (no first-class
admonitions, no callout-annotated listings, no conditional includes,
weak cross-references) starts to bite. Picking `asciidoctor` over
`pandoc` for AsciiDoc input is the standard recommendation upstream
even from the Pandoc maintainers, because Asciidoctor implements the
AsciiDoc Language Specification reference behaviour and Pandoc's
AsciiDoc reader is a best-effort approximation.

For LLM-CLI workflows, AsciiDoc's stricter grammar — required blank
lines around blocks, explicit section-level markers (`==` vs `===`),
typed admonitions (`NOTE:` / `WARNING:` / `IMPORTANT:` / `TIP:` /
`CAUTION:`) — produces less ambiguous parse trees than Markdown, which
matters when an agent is round-tripping documentation through an LLM
pipeline (parse → semantic edit → serialise) and Markdown's many
ambiguities (CommonMark vs GFM vs MultiMarkdown vs kramdown) tend to
mutate document structure across the round-trip. Pairs naturally with
`pandoc` (chain `asciidoctor -b docbook | pandoc -f docbook -t epub3`
for formats Asciidoctor does not target directly) and with `typst` on
the orthogonal axis (Typst for from-scratch typeset PDFs with a
modern markup grammar; Asciidoctor for inheriting and rendering the
existing AsciiDoc corpus).
