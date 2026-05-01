# newsboat

## What it does
A **terminal RSS / Atom feed reader** that fetches a flat list of feed
URLs (`~/.newsboat/urls`), stores items in a local SQLite cache, and
renders three stacked TUI panes — feed list, article list, article body
— navigated with vi-style keys. Supports tag-grouped feeds, query feeds
(SQL-shaped saved searches like `"unread == \"yes\" and age < 2"`),
podcast enclosures via the bundled `podboat` companion, OPML import /
export, OAuth-backed sync against Tiny Tiny RSS / Newsblur / FeedHQ /
Inoreader / Miniflux / FreshRSS / Owncloud-News, and a macro language
for piping the open article into arbitrary commands (open URL in
browser, save to Pocket, hand body to `pandoc`, etc.). Single C++ binary
plus the SQLite cache, no daemon, no server side, no telemetry.

## Why it's interesting
Different shape from web aggregators (Feedly, Inoreader web UI — needs
a browser, vendor lock-in, ads), from RSS-in-email gateways (Kill the
Newsletter — async, mailbox shape), from [`nb`](../nb/) / Obsidian-style
note tools (read-later, not real-time pull), from IRC / chat firehoses
(push-shape), and from the long-dead `snownews` / `canto` lineage
(unmaintained). Inside the catalog, [`frogmouth`](../frogmouth/) and
[`glow`](../glow/) read *Markdown* in the terminal, [`elia`](../elia/)
is a chat TUI, none of them poll the web for you. newsboat is the
*synchronous personal RSS client* shape: pick it specifically when the
ask is "give me one terminal pane, vi keys, that pulls 200 feeds, marks
them read locally, and lets me batch-open URLs in `$BROWSER` over SSH"
— or when you already run a self-hosted Tiny Tiny RSS / Miniflux /
FreshRSS server and want a TUI client that syncs read-state back to it.
Do **not** pick it for podcast-first workflows (use a real podcast app
or the bundled `podboat` only for occasional grabs), for full-text
search across years of archives (use Miniflux + its web UI), or for
multi-device mobile reading (this is a single-machine TUI).

## Niche category
Terminal RSS / Atom reader — three-pane TUI over a local SQLite cache
with tag groups, SQL-shaped query feeds, OPML import, and optional
sync against TT-RSS / Newsblur / Miniflux / FreshRSS.

## Repo
https://github.com/newsboat/newsboat

## Version pinned
`r2.43` (latest tagged release; newsboat ships as `rX.Y` git tags)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew
brew install newsboat

# Debian / Ubuntu
sudo apt install newsboat

# Fedora
sudo dnf install newsboat

# Arch
sudo pacman -S newsboat

# From source (Rust + C++ toolchain required)
git clone https://github.com/newsboat/newsboat
cd newsboat
make
sudo make install
```

## Usage examples
```sh
# Seed the feed list
mkdir -p ~/.newsboat
cat >> ~/.newsboat/urls <<'EOF'
https://lwn.net/headlines/rss      tech kernel
https://www.schneier.com/feed/     security
"query:Unread Recent:unread = \"yes\" and age < 7"
EOF

# Pull all feeds and open the TUI
newsboat -r

# Refresh in the background and exit (cron-friendly)
newsboat -x reload

# Import / export OPML
newsboat -i feeds.opml
newsboat -e > backup.opml

# Inside the TUI:
#   j / k     next / previous
#   Enter     open article
#   o         open URL in $BROWSER
#   N         mark unread
#   A         mark all read
#   r         reload current feed
#   R         reload all
#   /         search
#   q         back / quit

# Companion: download podcast enclosures queued by newsboat
podboat
```
