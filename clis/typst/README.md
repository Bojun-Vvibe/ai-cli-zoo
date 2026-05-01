# typst

## What it does
A **modern markup-based typesetting system** — single Rust binary, no
TeX distribution required — that compiles `.typ` source files to
PDF, PNG, or SVG with sub-second incremental builds and a real
on-watch live-preview mode (`typst watch`). Source syntax is a hybrid
of Markdown-style content shorthand (`= Heading`, `*bold*`, `$ a^2 +
b^2 $`) and a strongly-typed expression language for layout, styles,
and computation; `#let`, `#show`, `#set` rules let you script
document templates without learning the macro-expansion semantics
that make `\renewcommand` a debugging adventure. The compiler is
deterministic, hermetic (fonts and packages are resolved against
declared inputs, not the system TeX tree), and ships with a built-in
math typesetter that approaches LaTeX quality on common formulas.
A package ecosystem (`@preview/...` namespace) covers CV / thesis /
slides / chemfig / cetz drawing / glossaries; `typst init` scaffolds
from a template, `typst query` extracts structured data (e.g. all
headings, citations) from a compiled document for downstream
pipelines.

## Why it's interesting
Different shape from LaTeX (battle-tested 40-year ecosystem but
glacial incremental compile, opaque error messages, multi-GB TeX
Live install, macro language predates structured programming), from
ConTeXt (powerful but niche, also TeX-engine-bound), from Pandoc
(format converter — emits LaTeX or HTML, doesn't typeset
high-fidelity PDFs on its own), from SILE (similar "modern
typesetter" ambition but Lua-driven and smaller community), and from
`weasyprint` / `prince` (HTML+CSS to PDF — different mental model,
weaker math). typst is the *one Rust binary, sub-second incremental
PDF, real type-checked styling language* shape: pick it specifically
for new authoring projects (papers, theses, books, slide decks,
generated reports) where you want LaTeX-class output without LaTeX
itself, and especially when you need programmatic document generation
in CI (`typst compile invoice.typ` runs in a 30 MB container with no
TeX). Do **not** pick it when you must collaborate on an existing
LaTeX manuscript (no LaTeX import path), when your venue mandates a
specific LaTeX class file (journal templates), or when you need
features like `pdflatex`-grade microtypography that the spec hasn't
caught up on yet — pair with [`pandoc`](../pandoc/) at the
boundaries.

## Niche category
Typesetting system — Rust-native LaTeX alternative with a
type-checked styling language, hermetic compilation, and sub-second
incremental builds suited to CI document generation.

## Repo
https://github.com/typst/typst

## Version pinned
`v0.13.1` (latest tagged release as of 2026-05-01)

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install typst

# Cargo (any platform)
cargo install --locked typst-cli

# Pre-built binaries for darwin / linux / windows
# https://github.com/typst/typst/releases/tag/v0.13.1

# Nix
nix profile install nixpkgs#typst
```

## Usage examples
```sh
# Compile a document to PDF (one-shot)
typst compile thesis.typ thesis.pdf

# Live preview: rebuild on every save, refresh PDF viewer automatically
typst watch report.typ

# Pin a font directory (hermetic builds in CI without system fonts)
typst compile --font-path ./fonts paper.typ

# Scaffold a new project from a community template
typst init @preview/modern-cv:0.7.0 my-cv && cd my-cv && typst compile main.typ

# Extract structured data from a compiled document (e.g. for indexing)
typst query paper.typ "<heading>" --field body --one false
```

## Date added
2026-05-01
