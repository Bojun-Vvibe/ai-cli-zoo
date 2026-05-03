# marp-cli

- **Repo:** https://github.com/marp-team/marp-cli
- **Version:** v4.3.1 (2026-03-16)
- **License:** MIT ([LICENSE](https://github.com/marp-team/marp-cli/blob/main/LICENSE))
- **Language:** TypeScript (Node.js); ships single-file binaries via `pkg`
- **Install:** `brew install marp-cli` · `npm i -g @marp-team/marp-cli` · `docker run --rm -v "$PWD":/home/marp/app marpteam/marp-cli deck.md` · pre-built single-file binaries on the [releases page](https://github.com/marp-team/marp-cli/releases) for macOS / Linux / Windows

## What it does

`marp-cli` (`marp` on `PATH`) converts a Markdown deck into a presentation: HTML self-contained slides, PDF, PPTX, PNG / JPEG per-slide images, or a live preview server. The input is plain CommonMark / GFM Markdown plus a small directive language at the top of the file (`marp: true`, `theme: gaia`, `paginate: true`, `size: 16:9`, `class: invert`, etc.) and a slide separator (`---` on its own line). A 5-line file (`marp: true` + a title + `---` + a content slide) renders to a polished HTML deck or a printable PDF with one command. Built on the [Marpit](https://marpit.marp.app/) framework (the rendering engine — same authors), the CLI's surface stays small but is powerful when composed: `marp deck.md` writes `deck.html`; `marp --pdf deck.md` writes `deck.pdf` (using a headless Chromium under the hood, downloaded once and cached); `marp --pptx deck.md` writes a real `.pptx` with editable text frames so a colleague can finish editing it in PowerPoint / Keynote / Google Slides; `marp --images png deck.md` writes one PNG per slide for embedding into READMEs, blog posts, or social previews; `marp --server .` boots a local dev server that watches the directory and live-reloads in the browser as you type. Three built-in themes ship out of the box (`default`, `gaia`, `uncover`) and any CSS file can be a custom theme via `--theme my.css`. Scoped per-deck or per-slide HTML / CSS works, math via KaTeX (`$$x = y^2$$`) is on by default, Mermaid diagrams render via the same Chromium pipeline, and presenter notes (`<!-- _backgroundColor: black; _color: white -->` style directives plus speaker-note comment blocks) round out the surface. The CLI honors a `marp.config.js` / `marp.config.cjs` / `.marprc.yml` for project-wide defaults so a docs repo with 30 decks does not repeat the theme / footer / paginate flags everywhere.

## When to pick it / when not to

Pick `marp-cli` when slides are an artifact of an engineering repo and should live in version control next to the code they describe — design docs, RFC walk-throughs, internal tech-talks, conference submissions, workshop materials, weekly demo-day decks. Concrete cases: an RFC repo where each proposal has a `talk.md` you can render with one CI step into `talk.pdf` and `talk.html`, both attached to the PR for async review; a workshop repo that publishes one `slides/` directory and a CI job that runs `marp --html --pdf --images png slides/` to produce student-facing HTML, an offline-printable PDF, and per-slide PNGs for the README; a conference talk you want to keep as a real source artifact (Markdown) so 12 months later you can `git diff` what changed between v1 and the version you actually delivered; a docs site that embeds a 6-slide explainer as a `<iframe>` of a marp-rendered HTML deck. Pair with [`pandoc`](../pandoc/) when you also need DOCX / EPUB / LaTeX from the same Markdown source (pandoc handles general document conversion; marp specializes in slides); pair with [`presenterm`](../presenterm/) for a fully-terminal slide rendering that does not require Chromium (good for SSH-only demos); pair with [`slides`](../slides/) for a Charm-style minimal terminal deck without theme CSS; pair with [`patat`](../patat/) for a `pandoc`-backed terminal deck. Pair with [`mdbook`](../mdbook/) when the same content also wants to be a book.

Skip `marp-cli` when the deck is a one-off pitch with heavy custom layouts, animations, transitions, or designer-driven typography — a real slide tool (Keynote, PowerPoint, Figma Slides, Pitch) will be faster and the result will be richer; marp's Markdown-first model is deliberately constrained. Skip when your audience expects PowerPoint-native motion (build animations, slide transitions, embedded video timing); marp can produce `.pptx` but the output is intentionally simple and you lose the marp HTML feature parity. Skip when your environment forbids the headless Chromium download marp uses for `--pdf` / `--pptx` / `--images` (an air-gapped CI without `puppeteer`-friendly egress, for instance) — HTML-only output works without Chromium, but the rasterized formats do not. Skip when the slide content has zero engineering audience and lives in a Google Drive / Notion / Slides-native workflow already; converting the team to Markdown for a single deck is not worth it.

## Vs already cataloged

- **Vs [`presenterm`](../presenterm/):** different output surface. `presenterm` renders a Markdown deck inside the terminal — beautiful for SSH demos, language-syntax-highlighted code blocks via tree-sitter, no browser involved. `marp-cli` renders to HTML / PDF / PPTX / PNG for export and web-publishable decks. The same Markdown can often be authored to render acceptably in both, but the fit-and-finish differs.
- **Vs [`slides`](../slides/):** charm `slides` is a minimalist terminal deck runner with a small Markdown subset; marp-cli is the production-grade slide framework with theming, math, Mermaid, PDF / PPTX export, and live-reload dev mode. Pick `slides` for quickfire terminal demos with no setup, marp-cli for slides you keep around.
- **Vs [`patat`](../patat/):** patat uses pandoc as the renderer and runs the deck inside a terminal; marp uses Marpit + headless Chromium and exports portable artifacts. Different audiences (terminal demo vs published HTML/PDF).
- **Vs [`pandoc`](../pandoc/):** complementary. pandoc converts Markdown to many document formats including a basic Beamer / reveal.js slide output; marp-cli is slide-specialized (themes, slide directives, paginate, presenter mode, live server, per-slide PNG export). Use pandoc when you also need DOCX / EPUB / LaTeX from the same source; use marp when slides are the primary output.
- **Vs [`mdbook`](../mdbook/) / [`zola`](../zola/):** orthogonal. Those build static sites / books; marp builds slide decks. They share the "Markdown in, static asset out" pattern but produce different artifacts.
- **Vs [`vhs`](../vhs/) / [`asciinema`](../asciinema/):** orthogonal. Those record terminal sessions; marp produces slides. They often appear in the same talk repo (a `vhs` `.cast` embedded inside a marp-rendered HTML slide).

## Caveats

- **PDF / PPTX / image output requires a Chromium binary.** marp-cli uses puppeteer's bundled Chromium on first use; in a Docker-based CI, prefer the official `marpteam/marp-cli` image which already has Chromium installed and configured. Air-gapped environments need to mirror the Chromium download or stick to HTML-only output.
- **`.pptx` export is HTML-rasterized.** The PPTX file marp produces embeds rendered slide images plus a basic outline, *not* native PowerPoint shapes. Recipients can re-order or replace slides but cannot edit the per-shape text the way a Keynote-authored deck allows. If editable PPTX is required, author in Keynote / PowerPoint directly.
- **Themes are CSS.** Custom themes have full CSS power but inherit Marpit's slide layout model — if you want non-rectangular slide layouts or animated transitions, you are fighting the framework. Stick to the [theme authoring guide](https://marpit.marp.app/theme-css) and the result is great for content-first decks.
- **Markdown directives are HTML comments.** Per-slide settings live as `<!-- _class: invert -->` style comments. Some Markdown editors strip HTML comments on save (especially older Notion / Bear exports), which silently drops your slide-level styling. Author in a plain text editor or one whose Markdown mode preserves comments.
- **Math via KaTeX, not MathJax.** KaTeX is faster but supports a smaller TeX surface than MathJax. If a paper-quality math-heavy deck is the goal, you will hit unsupported macros; pandoc → Beamer is the more academic path there.
- MIT ([LICENSE](https://github.com/marp-team/marp-cli/blob/main/LICENSE)) — permissive; safe to use for commercial decks, internal training, and CI pipelines. The bundled Chromium follows its own (BSD-style) license; the marp themes ship under MIT as well.

## Example invocations

```bash
# Install
brew install marp-cli
npm i -g @marp-team/marp-cli

# Convert deck.md → deck.html (default)
marp deck.md

# PDF (headless Chromium under the hood; cached after first run)
marp --pdf deck.md

# PPTX (rasterized slides + outline; editable order, not editable shapes)
marp --pptx deck.md

# One PNG per slide (for README embeds, social previews)
marp --images png deck.md
ls deck.001.png deck.002.png ...

# Live-reload dev server while authoring
marp --server .
# now open http://localhost:8080 and edit deck.md — the browser refreshes

# Pick a theme
marp --theme gaia deck.md
marp --theme ./themes/company.css deck.md

# Allow embedded HTML / scripts (e.g. Mermaid via the marp-mermaid plugin)
marp --html deck.md

# Project-wide defaults via .marprc.yml
cat > .marprc.yml <<'YAML'
theme: gaia
options:
  looseYAML: false
allowLocalFiles: true
YAML
marp deck.md   # picks up the rc

# Docker (no local Node / Chromium required)
docker run --rm --init -v "$PWD":/home/marp/app -e LANG=$LANG \
  marpteam/marp-cli --pdf deck.md
```
