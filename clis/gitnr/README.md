# gitnr

- **Repo:** https://github.com/reemus-dev/gitnr
- **Latest version:** v0.3.0 (release `v0.3.0`)
- **HEAD on `main`:** `3305304`
- **License:** MIT — [LICENSE](https://github.com/reemus-dev/gitnr/blob/main/LICENSE)
- **Category:** dev productivity / project scaffolding

A **single-binary `.gitignore` generator** written in Rust that
composes a project's ignore file from any mix of four template
sources: the curated [`github/gitignore`](https://github.com/github/gitignore)
collection, the [Toptal `gitignore.io`](https://www.toptal.com/developers/gitignore)
service, an arbitrary remote URL, or a local file path. Templates
are referenced by short tokens (`gh:Rust`, `tt:macos`, `url:…`,
`file:…`) and merged in declaration order with **automatic dedup
of identical patterns**, so a polyglot repo can pull
`gh:Node gh:Python gh:Rust tt:macos,linux,visualstudiocode` in one
line and get a single coherent ignore file with no duplicate
`node_modules/` or `__pycache__/` blocks. The tool ships an
**interactive TUI selector** (`gitnr create`) that lists every
known template with fuzzy-search, lets you toggle entries, and
previews the merged output before writing it — useful when you
do not remember the exact GitHub template name for, say,
"Unity vs UnrealEngine vs Godot". Output goes to stdout by default
(pipeable into `tee .gitignore`) or to `--output` for direct write.

## Install

```bash
# macOS
brew install reemus-dev/tap/gitnr

# Cargo (any platform with a Rust toolchain)
cargo install gitnr

# Linux / macOS — prebuilt binary from a release
curl -L -o gitnr.tar.gz https://github.com/reemus-dev/gitnr/releases/download/v0.3.0/gitnr-x86_64-unknown-linux-gnu.tar.gz
tar xzf gitnr.tar.gz && sudo mv gitnr /usr/local/bin/

# verify
gitnr --version
```

## Usage examples

```bash
# Compose a .gitignore from the GitHub Rust + Node templates plus
# the Toptal macOS + VSCode templates, dedup, write to file:
gitnr create gh:Rust gh:Node tt:macos,visualstudiocode \
  --output .gitignore

# Interactive TUI — fuzzy-search the full template catalogue, toggle
# entries, preview the merged result, then write on confirm:
gitnr create
```

## Why it matters

Most repos accumulate a hand-edited `.gitignore` that is part GitHub
template, part stack-overflow paste, part muscle memory — with
duplicate rules, missing IDE entries, and no record of *which*
templates were ever pulled. `gitnr` replaces that drift with a
**reproducible one-liner**: the four-source token syntax is the
spec, the merged file is the artifact, and re-running the same
command after a year of upstream template churn gives you a
faithful refresh with one diff to review. It fits the same niche
that [`mise`](../mise/) fills for tool versions and that
[`chezmoi`](../chezmoi/) fills for dotfiles — declarative, source-
controllable, idempotent — but for the one file every repo has and
nobody curates well. Compared to copy-pasting from `gitignore.io`
in a browser, it is faster, scriptable from CI, and removes the
"two engineers, two slightly different ignore files" failure mode
on day-one of a new service.
