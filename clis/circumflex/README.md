# circumflex

> **Hacker News in your terminal, with proper Markdown rendering and
> a built-in comment reader.** A single Go binary that pulls the HN
> API directly, paginates the front page / new / show / ask, and
> opens stories or comment threads in a vi-keys TUI.
> Pinned to **v4.0**
> ([LICENSE](https://github.com/bensadeh/circumflex/blob/main/LICENSE),
> MIT).

Source: <https://github.com/bensadeh/circumflex>

## TL;DR

`circumflex` (CLI name: `clx`) is a TUI that turns the HN front page
into a paginated list and the comment threads into an indented,
syntax-highlighted reader. It hits the official Algolia + Firebase
HN endpoints, caches nothing on disk by default, and renders comment
Markdown (code blocks, italics, links) via `glamour`. The whole UI
is keyboard-driven: `j/k` to move, `o` to open the article in `$BROWSER`,
`Enter` or `c` to dive into comments. No login, no posting, no API
key — read-only by design.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install circumflex

# Arch (AUR)
yay -S circumflex

# Go
go install github.com/bensadeh/circumflex@latest

# Docker
docker run -it --rm bensadeh/circumflex

# verify
clx --version    # 4.0
```

## License

MIT — see
[LICENSE](https://github.com/bensadeh/circumflex/blob/main/LICENSE).
Permissive, no copyleft. Safe to bundle into a curated dotfiles
image or a developer-onboarding container.

## One Concrete Example

```bash
# Open the HN front page (default tab)
clx

# Inside the TUI:
#   j / k          move down / up
#   Enter or c     open the comment thread for the highlighted story
#   o              open the linked article in $BROWSER
#   r              refresh the list
#   1 / 2 / 3 / 4  switch tabs: front / new / ask / show
#   q              quit
#
# Inside a comment thread:
#   j / k          next / previous comment
#   h / l          collapse / expand subtree
#   g / G          jump to top / bottom
#   q              back to story list
```

Configuration lives at `~/.config/circumflex/config.env`; useful
keys: `comment_width=80`, `markdown_renderer=true`,
`hide_url_in_header=false`.

## Niche It Fills

**The "I want HN comments without leaving the terminal, and I want
them readable" gap.** The HN web UI is plain text; pasting it into
`w3m` or `lynx` gives you the markup but mangles thread indentation
and code blocks. `circumflex` is purpose-built for the comment-tree
shape: indentation matches reply depth, code blocks render with
syntax highlighting, and collapse/expand on a subtree means you can
skim a 600-comment thread in two minutes. Pairs naturally with a
keyboard-only workflow inside `tmux` / `zellij`.

## Why use it

1. **Comment-thread shape is first-class.** Most "HN in terminal"
   readers throw the comments at you as a flat list. `circumflex`
   indents by reply depth and lets you collapse a subtree with `h`,
   which is the only sane way to read big threads.
2. **Markdown renders correctly.** `glamour` handles code fences,
   inline code, italics, and links — so technical comments (which
   are most of HN) read like a blog post, not a raw text dump.
3. **Single static Go binary, read-only API.** No login, no token,
   no rate-limit dance, no daemon. One `brew install`, one `clx`,
   you're reading. Trivial to drop into a minimal container or
   ship to a junior dev's onboarding box.

## Vs Already Cataloged

- **Vs [`newsboat`](../newsboat/):** `newsboat` is a generic RSS /
  Atom reader; you can point it at HN's RSS feed and get titles,
  but you don't get the comment thread shape, collapsing, or
  Markdown rendering. `circumflex` is HN-shaped and only HN-shaped.
- **Vs a browser tab:** A browser tab gives you the full HN UI but
  pulls you out of the terminal context. `circumflex` keeps you in
  `tmux` so the "read HN for five minutes between builds" cadence
  doesn't break flow.
- **Vs `lynx https://news.ycombinator.com`:** `lynx` renders the
  HTML but the indentation and code blocks come out flat. No
  collapse, no syntax highlighting, no per-thread navigation.

## Caveats

- **Read-only.** No upvoting, no commenting, no submitting. By
  design — adding write needs OAuth and account state. Use the
  web for that.
- **No offline mode.** Each launch hits the HN API. On a flaky
  link the list takes a couple of seconds to populate; comments
  load lazily when you enter a thread.
- **HN-specific.** Not a generic forum reader. If you also want
  Lobsters / Reddit in the same UI, this isn't it.
- **Terminal must support 256-color + UTF-8** for the Markdown
  renderer to look right; on a stripped-down `TERM=xterm` the
  output degrades to plain ANSI.
