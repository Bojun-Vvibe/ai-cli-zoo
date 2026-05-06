# oranda

> **Generates a polished release / docs landing site for a CLI
> project from its `Cargo.toml` (or equivalent), with installer
> snippets, changelog, and download tables wired up
> automatically.** Pinned to **v0.6.5**
> ([LICENSE-APACHE](https://github.com/axodotdev/oranda/blob/main/LICENSE-APACHE),
> Apache-2.0; also dual-available under MIT via
> [LICENSE-MIT](https://github.com/axodotdev/oranda/blob/main/LICENSE-MIT)).

Source: <https://github.com/axodotdev/oranda>

## TL;DR

`oranda` is a static-site generator with one specific job:
turn a CLI release repository into a website that looks like
the kind of site `ripgrep` or `bat` would have if their
authors had time to build one. It reads metadata you already
have (`Cargo.toml` / `package.json` / `pyproject.toml`),
discovers your GitHub releases, formats your `CHANGELOG.md`,
and emits installer one-liners (Homebrew, `cargo install`,
`curl | sh` from cargo-dist, npm, pip, etc.) keyed to the
visitor's OS. Sister project to `cargo-dist` from axo.dev:
`cargo-dist` builds and signs the binaries; `oranda` builds
the page that hands them out. Project status: archived
upstream in 2024, but the v0.6.5 release is still widely
used and functions as documented; treat it as "stable but
unmaintained — pin and hold."

## Install

```bash
# Cargo (compiles from source)
cargo install --locked oranda

# Pre-built binary via cargo-binstall
cargo binstall oranda

# Direct curl-pipe (the project's own one-liner)
curl --proto '=https' --tlsv1.2 -LsSf \
  https://github.com/axodotdev/oranda/releases/download/v0.6.5/oranda-installer.sh | sh

# verify
oranda --version
```

## Examples

```bash
# In a repo with a Cargo.toml: zero-config dev preview
oranda dev               # serves on http://localhost:7400, hot-reloads on edits

# Build the static site for production into ./public
oranda build

# Generate a starter oranda.json so you can override defaults (theme,
# social cards, extra MD pages, multi-project workspace)
oranda config-schema > oranda.schema.json

# Wire it into GitHub Pages: a typical CI uses
#   - cargo-dist to cut the release artifacts
#   - oranda build to render the page
#   - actions/deploy-pages to publish ./public
```

## When to choose it

Pick `oranda` when you ship a **CLI** (or a small library
with a CLI) and you want a real landing page — install
instructions, changelog, screenshots, OS-aware download
table — without hand-rolling Hugo/Astro themes. The sweet
spot is "I already use `cargo-dist` (or could) and I want the
release page that goes with it." Concrete signals: a single-
binary CLI, a `CHANGELOG.md` you keep up to date, GitHub
Releases as your distribution channel, and no patience for a
full SSG.

Skip it for traditional documentation sites — long-form guides
with API reference, versioned docs, search across hundreds of
pages. Reach for [`mdbook`](../mdbook/),
[`docusaurus`](https://docusaurus.io/), or
[`zola`](../zola/) for those. Skip it too if your project
isn't archive-tolerant: upstream is no longer accepting
patches, so anything you'd want changed has to live in a fork.

## Vs adjacent tools

- **Vs [`mdbook`](../mdbook/):** `mdbook` is a generic book-
  shaped docs renderer; you bring the structure and write
  Markdown. `oranda` is opinionated about *one* page shape (a
  release landing page) and fills it in from your release
  metadata. Use `mdbook` for the Guide, `oranda` for the
  Home.
- **Vs Hugo / Astro / Eleventy / [`zola`](../zola/):**
  general-purpose SSGs that can render anything but require
  you to design the release-page shape yourself. `oranda` is
  the pre-built version of that one shape.
- **Vs `cargo-dist`:** sibling tool from the same authors.
  `cargo-dist` builds and signs the cross-platform release
  artifacts (tarballs, `.msi`, Homebrew tap PRs, shell-script
  installers). `oranda` renders the *page* that links to
  those artifacts. They are designed to be used together but
  work independently.
- **Vs GitHub's auto-generated release page:** GitHub gives
  you a flat list of release notes and asset names. `oranda`
  upgrades that to an OS-detected install snippet, a rendered
  changelog, and a project landing page on your own domain.
- **Vs [`goreleaser`](../goreleaser/) (Go ecosystem):**
  `goreleaser` overlaps with `cargo-dist` (it builds release
  artifacts and can publish a small Hugo-based site).
  `oranda` is the Rust-ecosystem-flavored equivalent of the
  page half — language-agnostic in input but Cargo-shaped in
  defaults.

## Caveats

- **Archived upstream.** The repo was archived in 2024. v0.6.5
  works as documented and many projects still ship it, but
  there will be no new features, no upstream bug fixes, and
  no security patches except via fork. Pin the version and
  audit when you bump the toolchain.
- **Best when paired with `cargo-dist`.** Without it, the
  installer-snippet feature loses much of its magic and you
  fall back to a static download list. Other ecosystems
  (npm, pip, Homebrew tap) work but with less metadata.
- **Single-page shape is the point.** If you find yourself
  fighting it to add a multi-section docs site, you've
  outgrown `oranda` — switch to a general SSG.
- **GitHub-centric.** Release discovery assumes the GitHub
  Releases API. GitLab / Forgejo / self-hosted git hosts work
  only via manual configuration.
