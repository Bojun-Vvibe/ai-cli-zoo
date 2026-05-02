# buku

> **Personal bookmark manager that treats your bookmarks as
> a private, portable, mergeable SQLite database** —
> everything is one `bookmarks.db` file, every record is
> URL + title + tags + description, and the CLI gives you
> import from every major browser, full-text and tag-based
> search, an optional Flask web UI, and a Python library
> API on top of it. Pinned to **v5.1**
> ([LICENSE](https://github.com/jarun/buku/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/jarun/buku>

## TL;DR

`buku` is a Python 3.10+ CLI bookmark manager whose entire
state is one SQLite file (default
`~/.local/share/buku/bookmarks.db` on Linux, equivalent
under XDG on macOS) — no cloud, no telemetry, no hidden
history, no analytics. The data model is deliberately small:
`(id, url, title, tags, description, immutable_flag)`. Around
that, the CLI gives you **the bookmark workflows that browser
extensions and SaaS bookmark managers usually monopolize**,
but local and scriptable: auto-fetch the title and meta
description when you add a URL (`-a`), batch-import from
Firefox / Chromium / Brave / Vivaldi / Edge with
`--ai` (auto-import, locks-aware), import/export to HTML /
XBEL / Markdown / Org / RSS / Atom / another buku DB,
multi-threaded full-DB title refresh (`-u` with `--threads
N`), regex search (`-r`), substring search (`--deep`),
field-prefixed search (`--markers` lets you write
`.title-substring :url-substring #tag` in one query), tag
boolean operators (`--stag 'kernel + debugging - books'`),
random-bookmark "rediscovery" mode (`--random N`), an
interactive sub-prompt with paged results, and Wayback-
Machine fallback (`--cached <id>`) when a link rots. Two
features make it a real long-term archive rather than a
toy: **immutable bookmarks** (`--immutable 1` pins the
title so a future site redesign can't overwrite your
curated label), and **redirect/error policies** for
batch refresh — `--url-redirect` follows permanent
redirects and rewrites the URL, `--tag-redirect 'http
redirect'` and `--tag-error 'http {}'` let you tag the rot
without losing the record, and `--del-error 400-404,500`
prunes truly dead links. There's also `bukuserver`, a Flask
web UI bundled in the same repo (run on `localhost` only;
the docs explicitly call it "personal use"), shell
completion for bash/fish/zsh, a man page, and a Python API
that other tools (Emacs `ebuku`, Firefox `bukubrow`,
`buku-dmenu`, `buku_run` rofi front-end) consume.

## Install

```bash
# PyPI (recommended)
pip3 install buku

# Homebrew (macOS / Linuxbrew)
brew install buku

# Arch
pacman -S buku

# Debian / Ubuntu (older versions; pip3 is fresher)
apt install buku

# From source pinned to release tag
git clone --depth 1 --branch v5.1 https://github.com/jarun/buku.git
cd buku && sudo make install
```

Optional integrations: `xsel` / `xclip` / `pbcopy` /
`wl-copy` for `c <id>` "copy URL to clipboard"; a `$VISUAL`
or `$EDITOR` for the editor-driven add/edit flow (`buku
-w`); `fzf` for the canonical fuzzy-pick:
`firefox $(buku -p -f 10 | fzf)`.

## Usage

```bash
# One-time: import everything from your browsers
buku --ai                                 # auto-import (close browsers first)
buku -i bookmarks.html                    # or import a Netscape HTML export

# Add a URL; title/description/tags are fetched from the page
buku -a https://example.org python, web tutorial -c "Best intro I've found"
buku -a https://ddg.gg search engine, privacy --title 'DDG' --immutable 1

# Add an editor-driven bookmark (uses $EDITOR / $VISUAL)
buku -w
buku -w 15012014                          # edit existing record by id

# Search
buku kernel debugging                     # any keyword
buku -S kernel debugging                  # all keywords
buku --deep pen                           # substring ('pen' matches 'opens')
buku -r '^https://docs\.'                 # regex
buku --stag 'rust + cli - book'           # rust AND cli, exclude book
buku --markers '.title :https #py,'       # title="title", url contains https, exact tag py

# Open / inspect
buku -p 100 200-210                       # print details for ids/ranges
buku -o 42                                # open id 42 in browser
buku --cached 42                          # open Wayback snapshot of id 42
buku --random 3                           # rediscover 3 random bookmarks

# Maintenance
buku -u                                   # re-fetch titles/descriptions for whole DB
buku -u --url-redirect --tag-redirect 'http redirect' \
        --tag-error 'http {}' --del-error 400-404,500
buku --replace 'old tag' 'new tag'        # bulk-rename a tag
buku -e bookmarks.html --stag 'rust'      # export filtered subset
buku -l 12                                # encrypt DB with 12 PBKDF2 iterations
buku -k 12                                # decrypt
```

## Why it's interesting

The bookmark-management slot is dominated by SaaS
(Pinboard, Raindrop, Pocket-was, etc.) and browser-native
sync — both of which lock you to a vendor and leak browsing
history as a side effect. The few local options usually
either (a) ride on a single browser's bookmark file (which
breaks the moment you switch browsers or want CLI access)
or (b) re-invent a giant Electron app. `buku` takes the
opposite trade: **one SQLite file, one CLI, one optional
local web UI, no network calls except on demand**. That
unlocks workflows the SaaS tools structurally can't:
`syncthing` your `bookmarks.db` between machines, `git`-
version a periodic export, pipe `buku -p -f 10` into `fzf`
for a millisecond launcher, write a 20-line Python script
against the library API to reorganize 5,000 bookmarks by
domain, or batch-revive dead links via Wayback. Pick `buku`
when (a) you have a multi-thousand-bookmark archive that
predates your current browser and you want it to outlive
the next browser swap, (b) you want bookmarks accessible
from a TTY / SSH session / dotfile-sync workflow, (c) you
want immutable curated titles that don't get clobbered when
sites redesign, or (d) you want a Pinboard-shaped workflow
without the vendor. Not the right pick when you need
real-time multi-device sync with conflict resolution out of
the box (you'll need to bolt on `syncthing`/`unison` and
accept "last write wins"), when you need a polished
read-it-later reading experience (use [`wallabag`](https://wallabag.org/)
or similar), or when you want native browser-extension UX
without any setup (the upstream `bukubrow` extension exists
but adds a moving part). Active project since 2015,
v5.1 cut December 2025, GPL-3.0 — the license is worth
flagging because if you build a hosted derivative you must
ship source. Compare with [`shiori`](../shiori/) (Go,
single-binary, ships its own web UI as the primary surface
and stores full HTML snapshots — better if you want
offline-first content archival, heavier if you just want a
URL list) and [`linkding`](../linkding/) (Django, designed
for self-hosted multi-user, requires a server) — `buku`
occupies the "single-file local DB, CLI-first, optional
web UI" slot they don't.
