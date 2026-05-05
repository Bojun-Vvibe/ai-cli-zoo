# wiki-tui

> **Wikipedia at the terminal** — a keyboard-driven TUI that
> searches Wikipedia, opens articles, follows wikilinks, and
> renders the prose with section folding, image placeholders,
> and configurable colour schemes — all without leaving the
> shell, and without a browser tab eating 400 MB of RAM.
> Pinned to **v0.9.2** (SPDX: `MIT`,
> [LICENSE](https://github.com/Builditluc/wiki-tui/blob/main/LICENSE)).

Source: <https://github.com/Builditluc/wiki-tui>

## TL;DR

`wiki-tui` is a single Rust binary (built on the `cursive`
TUI toolkit) that talks to the Wikipedia REST/Action API,
shows search results in a list view, and renders the chosen
article as wrapped, paginated text with clickable wikilinks
that open the linked article in the same session. Language
is configurable (`en`, `de`, `fr`, … any Wikipedia subdomain),
the colour scheme is themable, and the whole thing runs over
SSH on a server with no display, no browser, and no JavaScript
runtime.

## Install

```bash
# Homebrew
brew install wiki-tui

# Cargo
cargo install wiki-tui

# Pre-built binary (Linux / macOS / Windows)
# https://github.com/Builditluc/wiki-tui/releases/tag/v0.9.2

# verify
wiki-tui --version   # wiki-tui 0.9.2
```

## License

MIT — see
[LICENSE](https://github.com/Builditluc/wiki-tui/blob/main/LICENSE).

## Representative Commands

```bash
# 1. launch the interactive TUI; type a query, enter to search
wiki-tui

# 2. open straight to an article from the shell
wiki-tui "Rust (programming language)"

# 3. switch language at launch (German Wikipedia)
wiki-tui --language de "Bundestag"
```

Inside the TUI: `/` focuses search, `Enter` opens the
selected article, wikilinks are tab-cyclable and open in a
new view (Esc to pop back), `q` quits.

## Why It Matters

Wikipedia is the closest thing the internet has to a
universal reference, and most of the time the *prose* is what
you want — not the infobox sidebar, not the cookie banner,
not the eight tracking scripts. `wiki-tui` strips all of that
and gives you the article body in a colour-themable terminal
window, with wikilink navigation preserved so you can
rabbit-hole through "Carolingian Renaissance → Alcuin of York
→ trivium" without ever touching a mouse. It's the right tool
when you're on a remote box without a browser, when you're
deep in a `tmux` session and a context switch costs flow,
when bandwidth is metered, or when you just want the
encyclopedia without the web. Distinct from the catalog's
`tldr` family (command cheatsheets) and `glow` (offline
Markdown viewer): this is the live encyclopedia, online,
text-only, keyboard-only.
