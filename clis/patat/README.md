# patat

> **Terminal-based presentation tool that renders a Pandoc-
> parsed Markdown deck** — slides advance with the keyboard,
> embedded code blocks can be evaluated *live* through
> arbitrary interpreters (`ghci`, `python`, `bash`,
> anything), and the renderer reuses Pandoc so the same `.md`
> doubles as a paper / blog post. Pinned to **v0.15.2.0**
> ([LICENSE](https://github.com/jaspervdj/patat/blob/main/LICENSE),
> GPL-2.0-only).

Source: <https://github.com/jaspervdj/patat>

## TL;DR

`patat` (**P**resentations **A**top **T**he **A**NSI
**T**erminal) is a terminal presenter for the case where
the slide deck is a one-file Markdown document you already
have. Pandoc reads the file, `patat` walks the resulting
AST and renders one slide per `---` / `# Heading` boundary
into the terminal using truecolor + Unicode box-drawing.
Where it diverges from [`presenterm`](../presenterm/) /
[`slides`](../slides/) is **`eval`**: a fenced code block
tagged with a class can be piped through any external
command at presentation time, output captured inline, and
the result rendered as part of the slide — so a Haskell
talk runs `ghci` snippets live, a SQL talk runs a query
against `psql` and shows the result table, a shell talk
runs `bash` commands and you see the actual output, and
none of it is faked. The "incremental display" feature
re-renders the same code block on each keypress, so you can
walk through a 20-line program one line at a time
(`patat`'s built-in version of "fragments"). Speaker notes,
auto-advance with `--watch`, syntax highlighting via
`skylighting`, image rendering via `kitty` / `iterm2` /
`w3m` graphics protocols on capable terminals, slide-level
metadata in YAML front-matter — all in a single Haskell
binary that ships static-linked release artifacts.

## Install

```bash
# Homebrew
brew install patat

# Debian / Ubuntu
sudo apt install patat

# Single-binary download (GitHub releases)
curl -L -o patat.tar.gz \
  https://github.com/jaspervdj/patat/releases/download/v0.15.2.0/patat-v0.15.2.0-linux-x86_64.tar.gz
tar xzf patat.tar.gz && sudo mv patat /usr/local/bin/

# macOS arm64 ships as a zip
curl -LO https://github.com/jaspervdj/patat/releases/download/v0.15.2.0/patat-v0.15.2.0-darwin-arm64.zip
unzip patat-v0.15.2.0-darwin-arm64.zip && sudo mv patat /usr/local/bin/

# Build from source (Haskell, requires GHC + cabal)
git clone --depth 1 --branch v0.15.2.0 \
  https://github.com/jaspervdj/patat.git
cd patat && cabal install
```

## Usage

A minimal deck with a live-eval Haskell block:

````markdown
---
title: A patat talk
author: Me
patat:
  eval:
    ghci:
      command: ghci -ignore-dot-ghci
      fragment: false
      replace: true
---

# Why fold?

Folding is a thing.

```haskell {.eval pipe=ghci}
foldr (+) 0 [1..10]
```

---

# Bash works too

```bash {.eval}
date -u +%FT%TZ
```
````

```bash
# Run the deck (j / l / Space advance, k / h back, q quit)
patat talk.md

# Auto-reload on file change — edit slides in vim, see them update live
patat --watch talk.md

# Force-disable eval (safer when opening someone else's deck)
patat --force talk.md
```

## Why it's interesting

The terminal-presenter slot has live-evaluation as the
pivot question. [`presenterm`](../presenterm/) is the
modern Rust default with great defaults, image rendering,
and snippet execution — but its eval support is opinionated
about which interpreters to bundle. [`slides`](../slides/)
is Charm's minimalist take, optimized for visual polish,
with a much smaller eval surface. [`patat`](.) takes the
opposite trade: rendering is plainer, but **anything
shell-callable can be a live block** (`pipe=` declares the
command, `replace`/`fragment` controls the rerun loop), and
because rendering goes through Pandoc you get the full
Pandoc Markdown dialect (footnotes, citations, math via
unicode, tables, smart quotes) without the presenter
re-implementing a parser. Pick `patat` when (a) the talk
revolves around running real code in front of an audience
and you want REPL output to be the actual REPL not a
screenshot, (b) the slide source needs to *also* be a
publishable artifact (paper / README / blog post) without a
second toolchain, or (c) you live in Haskell-/Lisp-/Prolog-
adjacent communities where `patat` originated and the
incremental-fragments + ghci-replace combo is the standard
demo idiom. Not the right pick for media-heavy decks (no
animation, no transitions — for that, the slot is
[`presenterm`](../presenterm/) on a graphical terminal) or
for non-technical audiences who want PowerPoint-style
visuals. Stable, releases on a steady cadence since 2016,
maintained by the author of `hakyll` so the Pandoc
integration stays current.
