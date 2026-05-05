# hgrep

> **Human-friendly grep that prints results as syntax-highlighted
> code snippets with surrounding context**, complete with file
> headers, line numbers, and folded gutters. Built in Rust on top
> of `grep-regex` and `bat`'s syntax-highlighting machinery.
> Pinned to **v0.3.9** (SPDX: `MIT`,
> [LICENSE.txt](https://github.com/rhysd/hgrep/blob/main/LICENSE.txt)).

Source: <https://github.com/rhysd/hgrep>

## TL;DR

`hgrep` is what you reach for when `rg` gives you the right hits
but the output is a wall of `path:line:match` lines and you really
wanted to *read* the surrounding code. It runs a ripgrep-style
search, then for each match renders a small bat-style snippet
(syntax-highlighted, with line numbers and ±N lines of context)
collapsed by chunk. One scroll and you can answer "is this the
match I care about?" without opening the file.

## Install

```bash
# Homebrew (macOS / Linux)
brew install hgrep

# Cargo
cargo install hgrep --locked

# Pre-built release binary
curl -Lo hgrep.zip "https://github.com/rhysd/hgrep/releases/download/v0.3.9/hgrep-v0.3.9-aarch64-apple-darwin.zip"
unzip hgrep.zip
sudo install hgrep /usr/local/bin/

# verify
hgrep --version    # hgrep 0.3.9
```

## License

MIT — see
[LICENSE.txt](https://github.com/rhysd/hgrep/blob/main/LICENSE.txt).

## One Concrete Example

```bash
# 1. basic search with default 5-line context, syntax-highlighted
hgrep 'fn main' src/

# 2. larger context window
hgrep -c 12 'TODO' .

# 3. consume ripgrep JSON output (delegate the search to rg)
rg --json 'unsafe' | hgrep

# 4. only show matches inside Rust files
hgrep -G '*.rs' 'unwrap\(\)' .

# 5. tweak the theme (uses bat's syntect themes)
hgrep --theme 'Solarized (dark)' 'panic!' src/

# 6. printer mode for piping into a pager without escape codes
hgrep --no-grid --printer syntect 'TODO' . | less -R
```

## Niche It Fills

**The "I want grep results I can actually read" tier.** `rg` and
`grep` are tuned for *machine* consumption (one line per hit so
`xargs` works). `hgrep` is tuned for *human* triage: each hit
gets a real code snippet with context, language-aware highlighting,
and a header. It is the natural pairing for code review,
incident triage, or "what does this symbol look like in three
different files?" exploration.

## Why use it

1. **Snippets, not lines.** Every match comes with surrounding
   code in the same syntax theme `bat` would use, so you can
   judge relevance without opening the file.
2. **Pipes from `rg`.** If you already have a finely-tuned
   ripgrep invocation, just append `| hgrep` to upgrade the
   rendering. No need to learn a new query language.
3. **Foldable chunks.** Multiple hits in the same file are
   grouped under one header with a single set of separators,
   which keeps long result sets scrollable.
4. **Pure stdout.** No TUI, no alt-screen, no mouse — works
   inside `tmux`, `less -R`, CI logs, and shell pipelines.
5. **Fast.** Built on the same `grep-regex` crate that powers
   ripgrep; the highlighting is the only added cost.

## Vs Already Cataloged

- **Vs [`ripgrep`](../ripgrep/):** `rg` is the search engine;
  `hgrep` is the *renderer*. Use them together: `rg --json ... | hgrep`.
- **Vs [`bat`](../bat/):** `bat` highlights one file you already
  picked. `hgrep` highlights the *N* small windows that matched
  a query across many files.
- **Vs [`grep-ast`](../grep-ast/):** `grep-ast` uses tree-sitter
  to expand a hit to the enclosing function/class. `hgrep` uses
  fixed line context. Different mental model: `grep-ast` is
  semantic, `hgrep` is visual.
- **Vs [`igrep`](../igrep/):** `igrep` is an interactive TUI for
  iterative refinement. `hgrep` is one-shot and pipe-friendly.

## Caveats

- **Highlighting costs CPU.** On a multi-thousand-hit search, the
  syntect pass is noticeably slower than raw `rg`. Narrow the
  search first.
- **Theme selection lives in bat-land.** `hgrep --list-themes` to
  see what is available; the names match `bat`'s.
- **Not a structural search.** If you want "find this hit only
  inside the body of `fn foo`", reach for `ast-grep` or
  `grep-ast` instead.
- **Default printer is `syntect`.** On terminals that mis-detect
  truecolor support, force `--printer bat` or set `COLORTERM`.
