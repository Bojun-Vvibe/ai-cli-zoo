# hugo

> **The fast Go static-site generator: one binary builds a 10,000-page
> site in seconds from Markdown + YAML/TOML/JSON front-matter + Go
> templates, with first-class taxonomies, multilingual sites, asset
> pipelines (Sass / PostCSS / image processing / fingerprinting),
> live-reload dev server, and zero runtime dependency at deploy
> (output is plain HTML + CSS + JS).** Pinned to **v0.161.1**,
> Apache-2.0
> ([LICENSE](https://github.com/gohugoio/hugo/blob/master/LICENSE)).

- **Repo:** https://github.com/gohugoio/hugo
- **Latest version:** v0.161.1 (2026-04-29)
- **License:** Apache-2.0 (`LICENSE` at repo root)
- **Category:** `static-site-generator` / `docs` / `blog` / `markdown-publishing`
- **Language:** Go

## What it does

`hugo` reads a directory tree of Markdown files (and optional
`.html` / `.adoc` / `.org` / `.pandoc` content), parses front-matter
(YAML, TOML, or JSON), applies Go `html/template` (or Amber, or
Ace) templates organised by content "section" + "kind" + "type"
through a documented lookup chain, and writes a fully-baked static
site to `public/`. The build is single-pass and aggressively
parallel: a 10,000-page documentation site routinely builds in
under 10 seconds on a laptop, which is the property that
distinguishes hugo from Jekyll (Ruby, slow), Gatsby (Node + React
SSR, slow + flaky), and Eleventy (Node, faster but still
single-threaded). Built-in features the comparable tools either
lack or treat as plugins: full-text search via output of a
client-side index (`outputs: [HTML, JSON]`), taxonomies (tags +
categories + arbitrary user-defined facets) with auto-generated
listing pages and RSS feeds per facet, multilingual sites with
translation linkage between content files, image processing with
on-disk caching (`{{ $img := resources.Get "hero.jpg" }}{{ $img.Resize "800x" }}`),
asset bundling + minification + fingerprinting + Subresource
Integrity hashes, Sass/SCSS via embedded `dart-sass` or LibSass,
shortcodes for reusable parameterised content blocks, and a
`hugo server` dev mode with sub-millisecond live-reload via
filesystem watch + WebSocket. Output is the simplest possible
deploy artefact: a static directory served by any HTTP server,
CDN, or object store.

## Install

```bash
# macOS / Linux via Homebrew (the "extended" variant includes dart-sass + WebP)
brew install hugo

# Linux package managers
apt install hugo                   # Debian / Ubuntu
pacman -S hugo                     # Arch
dnf install hugo                   # Fedora

# Windows
choco install hugo-extended
scoop install hugo-extended

# Go install (always builds the extended variant against current Go)
go install -tags extended github.com/gohugoio/hugo@latest

# Pre-built binaries on every release
# https://github.com/gohugoio/hugo/releases
```

## Examples

```bash
# Bootstrap a new site
hugo new site myblog
cd myblog

# Add a theme (modules system; pulls a versioned go.mod-style entry)
hugo mod init github.com/me/myblog
echo 'theme = "github.com/adityatelange/hugo-PaperMod"' >> hugo.toml
hugo mod get -u

# Create a new post (front-matter + skeleton from archetype)
hugo new content posts/first-post.md

# Live-reload dev server with draft + future + expired posts visible
hugo server -D -F -E --bind 0.0.0.0

# Production build (minified, no draft content, with build stats)
hugo --minify --gc --logLevel info

# Build only a subset (single section) for fast preview
hugo server --renderToMemory --disableFastRender

# Multilingual site: build English + Japanese + Spanish in one pass
hugo --gc                          # config.toml declares all three

# Deploy hook: build then sync to S3 / GCS / Azure with the deploy provider
hugo deploy --target production --maxDeletes 50
```

## Why it matters in an AI-native workflow

Agent-generated technical content (release notes, API references,
runbook indexes, weekly digests) lands as Markdown by default —
which means the publishing layer either is, or pretends to be, a
Markdown-to-HTML pipeline. The two failure modes the agent loop
amplifies are (1) build time: when an agent regenerates 200 pages
overnight, a Jekyll / Gatsby site that took 4 minutes per build
now takes 4 minutes per *iteration*, breaks the inner loop, and
forces the agent to ship blind; and (2) brittleness: Node-based
generators with deep plugin chains break when a transitive
dependency yanks, and the agent cannot tell whether its content
edit or `npm`'s tree shake broke the build. `hugo` collapses both
problems: the build is sub-10s for sites large enough that the
agent's edit batch is *small* relative to the corpus, and the
"runtime" is one statically-linked Go binary with no plugin
manager and no `node_modules`. Pairs with [`mdbook`](../mdbook/)
(mdbook for *book*-shaped docs with one linear `SUMMARY.md`
ordering — narrow + deep; hugo for *site*-shaped collections with
sections + tags + taxonomies — broad + shallow; the two are
orthogonal and many orgs run both), [`zola`](../zola/) (zola is
the Rust analogue with a smaller feature surface and Tera
templates instead of Go templates — pick zola when "smaller and
opinionated" matters more than "wider plugin/theme ecosystem",
hugo when the opposite is true), [`marker`](../marker/)
(marker → Markdown ingestion of legacy PDFs → hugo build to
publish), [`glow`](../glow/) (glow renders the same Markdown
to a terminal, useful for previewing posts inside an SSH session
before `hugo server` is reachable), and complementary to
[`pandoc`](../pandoc/) (pandoc transforms one document; hugo
builds an entire site of cross-linked documents).
