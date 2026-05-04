# ddgr

> **DuckDuckGo from the command line** — a single-file Python
> script that issues DuckDuckGo searches without a browser, parses
> the result page, and renders a numbered, navigable list directly
> in the terminal. No API key, no tracking cookies, no JavaScript
> runtime. Pinned to **v2.2** (released 2023-09-30,
> [LICENSE](https://github.com/jarun/ddgr/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/jarun/ddgr>

## TL;DR

`ddgr` ("duck-duck-go-er") is the spiritual sibling of `googler`
but for DuckDuckGo. Run `ddgr "rust async traits"` and you get
the top results as a numbered list with title, URL, and snippet.
Then you can press `n`/`p` for next/previous page, type a number
to open that result in `$BROWSER` (or `o N` to open multiple),
type `c N` to copy the URL, type a new query inline to refine,
or pipe results to `jq` with `--json` for scripting. Bang
operators (`!w`, `!gh`, `!so`) work the same way they do on the
DuckDuckGo site, so `ddgr "!w halting problem"` jumps straight
to Wikipedia.

The whole tool is one self-contained Python file with zero
runtime dependencies beyond the stdlib — drop it on a server
and you have web search over SSH.

## Why it's interesting

For research-heavy terminal workflows (writing docs, debugging
an obscure stack trace, looking up an RFC) the browser context
switch is *the* productivity tax. `ddgr` collapses search →
read snippet → open chosen result into a single tmux pane,
keeps your fingers on the keyboard, and — uniquely — does it
against an engine that does not require an account, an API key,
or a cookie jar. Because it is one Python file with no deps it
is also the easiest "web search inside an LLM agent" primitive
for shell-based agents that already have Python available.

## Install

```bash
# macOS
brew install ddgr

# Debian / Ubuntu
sudo apt install ddgr

# Arch
sudo pacman -S ddgr

# pip (user-local)
pip install --user ddgr

# or just download the single file
curl -fsSL https://raw.githubusercontent.com/jarun/ddgr/v2.2/ddgr -o ~/.local/bin/ddgr
chmod +x ~/.local/bin/ddgr

# verify
ddgr --version    # 2.2
```

## Examples

```bash
# basic search — opens an interactive numbered prompt
ddgr "rust pinning explained"

# non-interactive: print top 5 results as JSON, pipe to jq
ddgr --num 5 --json "sqlite wal mode" | jq '.[].url'

# region-locked, time-filtered search (past month, US English)
ddgr -r us-en --time m "openssl 3.0 deprecations"

# DuckDuckGo bang: search Wikipedia directly
ddgr "!w category theory"

# search GitHub via the !gh bang
ddgr "!gh ripgrep"

# omit URLs, just titles + snippets (good for piping to less)
ddgr --omit-titles=false --no-prompt --num 10 "kubernetes leader election"

# inside the interactive prompt: open results 1, 3, 5
# > o 1 3 5
# next page
# > n
# refine query
# > rust async runtime comparison
```

## Use when

- You want web search inside tmux / SSH without launching a
  browser (and without an API key).
- You are scripting "look something up, then act on the URL"
  pipelines and need machine-readable output (`--json`).
- You build LLM tool-using agents on a shell host and want a
  single-file, dependency-free web-search tool you can `exec`
  with full control over the result format.
- You want bangs (`!w`, `!gh`, `!so`, `!arch`) as a fast jump
  table to Wikipedia, GitHub, Stack Overflow, ArchWiki, etc.

Skip `ddgr` if you need Google-specific results (use
[`googler`](https://github.com/jarun/googler) — same author,
same UX), if you need full HTML rendering (use
[`browsh`](https://github.com/browsh-org/browsh) or
[`carbonyl`](https://github.com/fathyb/carbonyl)), or if your
search engine of choice is paywalled / API-only (Kagi, Brave
Search Premium).
