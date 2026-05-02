# tuifeed

> **A keyboard-only RSS / Atom reader that lives in one terminal
> window.** A single Rust binary built on `tui-realm`: edit a TOML
> source list, launch `tuifeed`, get a three-pane (sources / posts
> / article) reader with vi-keys and inline article rendering.
> Pinned to **v0.4.2**
> ([LICENSE](https://github.com/veeso/tuifeed/blob/main/LICENSE),
> MIT).

Source: <https://github.com/veeso/tuifeed>

## TL;DR

`tuifeed` is a self-contained TUI feed reader: no daemon, no sync
service, no local SQLite — just one TOML config file listing your
feeds, and a three-pane UI that fetches them concurrently on launch
and on demand. The middle pane lists posts for the highlighted
source; the right pane renders the selected article's HTML body
into terminal-readable text via `html2text`. `o` opens the original
URL in `$BROWSER`; everything else is keyboard.

## Install

```bash
# Cargo
cargo install tuifeed

# Homebrew (macOS / Linuxbrew)
brew install tuifeed

# Pre-built binaries
# https://github.com/veeso/tuifeed/releases/latest

# verify
tuifeed --version    # tuifeed 0.4.2
```

## License

MIT — see
[LICENSE](https://github.com/veeso/tuifeed/blob/main/LICENSE).
Permissive, embed-friendly. Safe to drop into a developer image
without legal review.

## One Concrete Example

```bash
# 1. Edit the source list (created on first launch)
$EDITOR ~/.config/tuifeed/config.toml
```

```toml
# ~/.config/tuifeed/config.toml
[sources]
"Hacker News"     = "https://news.ycombinator.com/rss"
"Lobsters"        = "https://lobste.rs/rss"
"LWN headlines"   = "https://lwn.net/headlines/rss"
"Rust blog"       = "https://blog.rust-lang.org/feed.xml"
"Julia Evans"     = "https://jvns.ca/atom.xml"
```

```bash
# 2. Launch
tuifeed

# Inside the TUI:
#   ┌ Sources ──┬ Articles ─────────────┬ Body ──────────────────┐
#   │ Hacker N.. │ Show HN: I built ...  │ Title                  │
#   │ Lobsters   │ Ask HN: how do you... │                        │
#   │ LWN        │ ...                   │ rendered article text  │
#   └────────────┴────────────────────────┴────────────────────────┘
#
#   Tab            move between panes
#   j / k          next / previous item
#   Enter          load article into the body pane
#   o              open original URL in $BROWSER
#   r              refresh the highlighted source
#   R              refresh all sources
#   q              quit
```

## Niche It Fills

**The "I want RSS without a daemon, a database, or a sync service"
gap.** Self-hosted readers (Tiny Tiny RSS, FreshRSS, Miniflux) want
a server, a database, and ongoing maintenance. Mobile sync readers
(Feedly, Inoreader) want an account. Heavyweight TUIs
(`newsboat`) keep a local SQLite cache and an OPML config that has
to be kept in sync with reality. `tuifeed` is the smallest possible
slice: one TOML file, fetch on launch, render in three panes,
done. If you want "RSS in five minutes on a fresh machine", this
is the shortest path.

## Why use it

1. **Config is one TOML file.** `[sources]` is a plain string-to-URL
   map. Diff-friendly, dotfile-friendly, no OPML import dance.
   Adding a feed is one line; you can keep the file in version
   control and `stow` it across machines.
2. **Three-pane layout matches how feed reading works.** Most TUI
   readers force you to bounce between a list view and an article
   view modally. `tuifeed`'s three-pane layout means the article
   is always visible while you scan headlines — closer to a desktop
   reader's UX, no mode switching.
3. **No state, no migration risk.** No SQLite schema to upgrade,
   no daemon to restart, no read/unread sync (it's stateless within
   a session). Reinstall and your config still works.

## Vs Already Cataloged

- **Vs [`newsboat`](../newsboat/):** `newsboat` is the heavyweight:
  SQLite cache, podcasts, macros, query feeds, real read/unread
  state across runs, OPML import/export. Pick `newsboat` if you
  want a *durable* feed-reading workflow with hundreds of
  sources. Pick `tuifeed` if you want a *throwaway* reader on a
  fresh box with no state to migrate.
- **Vs a browser tab on Feedly:** Feedly is a SaaS account with
  a server-side read state. `tuifeed` keeps you in the terminal
  with no account anywhere.
- **Vs `curl <feed> | xmlstarlet`:** Possible but tedious; you
  reinvent the three-pane UI every time. `tuifeed` is the
  pre-built version.

## Caveats

- **Stateless within a session.** No persistent read / unread; on
  every launch every post starts as "unseen". Acceptable for
  small source lists; painful past ~30 feeds. Reach for `newsboat`
  in that case.
- **HTML-to-text only.** Article bodies render via `html2text`; no
  images, no embedded video, no JS. Code blocks survive, layouts
  do not. Press `o` to bounce the article into a real browser
  when you need rich rendering.
- **Sequential refresh on `R`.** Concurrent fetch helps, but if one
  source hangs (DNS timeout, slow server) the whole refresh waits
  for it. Trim the source list, or `r` (single-source refresh)
  instead of `R` (all).
- **Small project surface.** Maintained but low-traffic. Don't
  expect rapid feature additions; what's there is what you get.
- **Atom + RSS only.** No JSON Feed support; no podcast enclosure
  handling. If you mix podcasts in, use `newsboat` for those.
