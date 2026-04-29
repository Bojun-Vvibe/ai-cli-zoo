# mdbook

> **A static-site generator for book-shaped technical docs** — a single
> Rust binary that takes a tree of CommonMark `.md` files plus a
> `SUMMARY.md` table-of-contents and emits a self-contained, themeable,
> client-side-searchable HTML book. Pinned to **v0.5.2** (commit
> `7b29f8a7174fa4b7b31536b84ee62e50a786658b`,
> [LICENSE](https://github.com/rust-lang/mdBook/blob/master/LICENSE),
> MPL-2.0).

Source: <https://github.com/rust-lang/mdBook>

## TL;DR

`mdbook` is what `gitbook` was before it pivoted away from CLI users:
write your docs as ordinary Markdown files in a `src/` directory, list
them in `src/SUMMARY.md` (the only "magic" file — its nested bullet list
is the sidebar), run `mdbook build`, and get a static `book/` directory
with paginated pages, prev/next links, a sidebar, a search index, and
a dark/light theme toggle. The Rust ecosystem itself is the
existence proof: *The Rust Programming Language*, *The Rust Reference*,
*The Cargo Book*, *The Rustonomicon*, and *The Embedded Rust Book* are
all `mdbook` builds. There is no template DSL beyond CommonMark + a
handful of `{{#include}}` / `{{#playground}}` / `{{#rustdoc_include}}`
preprocessor directives, and the whole output bundle (HTML + CSS + JS +
search index) is one tarball you can drop on any static host (GitHub
Pages, S3, `python -m http.server`).

## Install

```bash
# Homebrew (macOS / Linux)
brew install mdbook

# Cargo
cargo install --locked mdbook

# Linux package managers
# Arch:   pacman -S mdbook
# Nix:    nix-env -iA nixpkgs.mdbook

# from a release tarball (any OS)
curl -Lo mdbook.tar.gz "https://github.com/rust-lang/mdBook/releases/download/v0.5.2/mdbook-v0.5.2-aarch64-apple-darwin.tar.gz"
tar xf mdbook.tar.gz
sudo install mdbook /usr/local/bin/

# verify
mdbook --version    # mdbook v0.5.2
```

## License

MPL-2.0 — see
[LICENSE](https://github.com/rust-lang/mdBook/blob/master/LICENSE).
File-level copyleft: changes to existing `mdbook` source files must be
released under MPL-2.0; your *book content* and any new files in your
own repo are unaffected. Safe to use in commercial / closed-source docs
pipelines.

## One Concrete Example

```bash
# 1. scaffold a new book
mdbook init my-handbook
cd my-handbook
# creates: book.toml  src/SUMMARY.md  src/chapter_1.md

# 2. live preview while you edit (rebuild + browser reload on save)
mdbook serve --open
# → http://localhost:3000

# 3. one-shot static build
mdbook build
# → ./book/  (deploy this directory anywhere)

# 4. include a code file verbatim, pinned to a line range,
#    so docs and source never drift
echo '{{#include ../examples/hello.rs:5:20}}' >> src/chapter_1.md

# 5. run all rust code blocks in the book as doctests
mdbook test

# 6. clean rebuild with a custom theme
mdbook build --dest-dir public/

# 7. typical book.toml additions
cat >> book.toml <<'TOML'
[output.html]
git-repository-url = "https://github.com/me/my-handbook"
edit-url-template  = "https://github.com/me/my-handbook/edit/main/{path}"
default-theme      = "ayu"

[output.html.search]
enable        = true
limit-results = 30
TOML
```

## Niche It Fills

**Long-form, paginated, multi-chapter technical docs as the unit of
output.** `mdbook` sits between (a) single-page Markdown READMEs (too
small once you have >5 sections), (b) wiki engines (require a server,
mutable state, accounts), and (c) general static-site generators like
Hugo / Zola / Jekyll (theme-and-config heavy, blog-shaped). The
"book" abstraction — ordered chapters, persistent sidebar, prev/next
nav, in-book search — is exactly right for a language reference, an
internal handbook, an SDK guide, or a multi-tutorial training site.
The whole pipeline is one binary, one TOML config, one `SUMMARY.md`,
and CommonMark; nothing to host, nothing to upgrade, no JS framework.

## Why use it

Three things `mdbook` does that justify picking it over a generic SSG:

1. **`SUMMARY.md` is the source of truth for structure.** The sidebar,
   chapter ordering, prev/next links, and even the breadcrumb are all
   derived from one nested-bullet Markdown file. Reorganising the book
   means editing one file; no front-matter `weight:` fields scattered
   across 80 chapters, no "category" taxonomy to maintain. Reviewers can
   diff `SUMMARY.md` to see exactly what changed in the book's
   structure.
2. **`{{#include}}` + `mdbook test` keep code samples honest.**
   Snippets are pulled from real source files (`{{#include
   ../src/lib.rs:42:60}}`) so the docs cannot drift past the line range
   without a build error, and `mdbook test` actually compiles + runs
   every Rust code block in the book against your project. Docs that
   compile are docs that stay correct.
3. **Output is a static-by-construction tarball.** The `book/` dir is
   pure HTML + CSS + a small JS bundle for search and theme; no Node
   runtime, no PHP, no server. Deploy with `aws s3 sync`, GitHub Pages,
   `caddy file-server`, or `python -m http.server` — same artifact, same
   behaviour, offline-readable, archive-friendly.

For an LLM-CLI workflow, `mdbook build` is a deterministic step you can
chain after a code-generation pass: agent edits `src/*.md`, `mdbook
build` produces a checkable artifact, `mdbook test` fails fast if a
generated code sample stopped compiling.

## Vs Already Cataloged

- **Vs [`marker`](../marker/):** opposite direction. `marker` converts
  PDF / EPUB / DOCX **into** Markdown so you can feed it to other CLIs;
  `mdbook` converts a Markdown tree **into** a paginated HTML book for
  humans. Pair them: use `marker` to ingest a legacy PDF spec, edit the
  resulting Markdown, then `mdbook build` to publish the modernised
  version.
- **Vs [`glow`](../glow/) / [`mdcat`](../mdcat/):** orthogonal. Those
  render Markdown in your *terminal* for one-off reading; `mdbook`
  renders a *whole tree* of Markdown into a *browser-shaped* book with
  navigation, search, and persistent URLs. Use `glow` for "let me read
  this README", use `mdbook` for "let me publish this 200-page handbook".
- **Vs `hugo` / `zola` / `jekyll` (general static-site generators, not
  cataloged):** those are blog/site shaped — taxonomies, archives,
  RSS, theme ecosystems. `mdbook` is book shaped — chapters, sidebar,
  prev/next, in-book search. If your content is "a sequence of pages
  meant to be read in order or jumped to via a TOC", `mdbook` removes
  90% of the config you would write in Hugo to get the same UX.
- **Vs MkDocs / Sphinx (Python ecosystem, not cataloged):** comparable
  scope, different tradeoffs. MkDocs/Material has a richer theme
  ecosystem and Sphinx has unmatched Python API doc autogeneration;
  `mdbook` wins on install footprint (one ~6 MB Rust binary, zero
  Python / Node dependencies), build speed (sub-second rebuilds for
  100-chapter books), and the no-template-DSL discipline. Pick MkDocs
  if you want the Material theme; pick Sphinx for Python autodoc; pick
  `mdbook` when you want CommonMark + a sidebar + search and nothing
  else to maintain.

## Caveats

- **Theming is intentionally limited.** There is one built-in theme
  with light / dark / ayu / rust / coal / navy variants and a
  customisable `theme/` directory you can override piece-by-piece, but
  you do not get a Hugo-style theme marketplace. If brand-perfect
  pixel control matters more than time-to-publish, this is a
  constraint.
- **Search is client-side and JS-only.** The build emits a JSON index
  loaded by an in-page JS searcher; large books (>1000 pages) can
  produce a multi-megabyte index that delays first-paint. There is no
  server-side / Algolia integration in core — community plugins exist
  but are out-of-tree.
- **Preprocessor surface is small.** `{{#include}}`, `{{#playground}}`,
  `{{#rustdoc_include}}`, and link / index preprocessors cover the
  Rust-docs use case; for Mermaid diagrams, KaTeX, admonitions, etc.
  you install third-party preprocessor binaries (`mdbook-mermaid`,
  `mdbook-katex`, `mdbook-admonish`) — supported but a separate
  install per feature.
- **No incremental build cache.** `mdbook build` re-renders the whole
  book on every invocation; `mdbook serve` is fast because it stays
  resident, but CI builds of large books pay the full cost each time.
  Workaround: persist the `book/` dir between CI runs and `rsync`
  only changed files to the host.
- **HTML is the only first-class output.** A bundled experimental
  Markdown renderer ("identity output") and community PDF / epub
  preprocessors exist, but the supported, polished pipeline is
  Markdown → HTML. If you need Markdown → print-quality PDF as a
  primary deliverable, pair with `pandoc` instead.
