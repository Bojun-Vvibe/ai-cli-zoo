# notmuch

> **Maildir indexer + tag-based mail search engine + minimal CLI
> driver** — points at a local Maildir tree, builds a Xapian
> full-text index over every message, and exposes Gmail-style
> tag-and-search semantics (`tag:inbox and from:alice and
> subject:invoice`) at sub-100ms speed across millions of
> messages. The CLI is the data plane: `notmuch new` (index),
> `notmuch search` (query), `notmuch show` (read), `notmuch
> tag` (mutate). Front-ends (emacs, [`alot`], [`astroid`],
> [`afew`], `notmuch-mutt`, [`aerc`](../aerc/)) reuse the same
> index. Pinned to **v0.40** (released 2026-04-23,
> [COPYING](https://github.com/notmuch/notmuch/blob/0.40/COPYING),
> GPL-3.0+).

Source: <https://github.com/notmuch/notmuch> (mirror of
<https://git.notmuchmail.org/git/notmuch>)

## TL;DR

Mail clients on a personal machine fall into three buckets:
(a) heavyweight IMAP-talking GUIs (Thunderbird, Apple Mail) —
fine until your archive crosses 200k messages and search becomes
glacial; (b) cloud-only webmail — Gmail / Fastmail give
instant search but leave your archive in a vendor's hands;
(c) the unix mail toolchain — sync IMAP to a local Maildir
with `mbsync` / `offlineimap`, index with `notmuch`, read with
whichever front-end you prefer. `notmuch` is the *index +
query* layer of (c): a Xapian-backed inverted index that turns
"all mail from Alice last quarter mentioning the contract" into
a 50ms query against a 2 million-message archive. The CLI is
deliberately minimal — `new`, `search`, `show`, `tag`, `count`,
`reply`, `dump`, `restore` — because the rich UX lives in the
front-end of your choice. Tags are first-class (no folders
required); a single message can carry `inbox`, `unread`,
`signed`, `customer/acme`, `project/q3-launch`. The on-disk
format is plain Maildir, so any other tool that reads Maildir
keeps working.

## Install

```bash
# macOS (Homebrew)
brew install notmuch

# Debian / Ubuntu
sudo apt-get install -y notmuch

# Arch
sudo pacman -S notmuch

# Fedora
sudo dnf install notmuch

# From source (release tarball)
curl -LO https://notmuchmail.org/releases/notmuch-0.40.tar.xz
tar -xJf notmuch-0.40.tar.xz && cd notmuch-0.40
./configure && make && sudo make install

# Verify
notmuch --version          # notmuch 0.40
```

First-run setup — point at a Maildir, set identity:

```bash
notmuch setup              # interactive: path, name, email, tags
# or write ~/.notmuch-config directly:
cat > ~/.notmuch-config <<'EOF'
[database]
path=~/mail
[user]
name=Alice Example
primary_email=alice@example.org
[new]
tags=unread;inbox;
EOF

notmuch new                # initial index (slow once, fast forever after)
```

## License

GPL-3.0+ — see
[COPYING](https://github.com/notmuch/notmuch/blob/0.40/COPYING).
Copyleft: distributing modified `notmuch` requires the same
licence. The Xapian backend is also GPL; query results /
database contents are your own data and unencumbered.

## Common invocations

```bash
# Index — run after every mail sync
notmuch new                 # incremental; safe to cron every minute

# Search — Gmail-shaped query language
notmuch search tag:inbox and tag:unread
notmuch search 'from:alice@example.org and date:2026-01-01..2026-05-31'
notmuch search 'subject:"quarterly review" and not tag:archived'
notmuch search 'thread:{from:bob and to:alice} and attachment:pdf'

# Show — full thread, decoded MIME
notmuch show --format=text id:20260504123456.foo@example.org
notmuch show --format=json thread:0000000000123abc

# Tag — bulk mutation by query
notmuch tag +inbox -unread -- tag:inbox and tag:unread
notmuch tag +customer/acme -- 'from:@acme.example and date:2026-01-01..'
notmuch tag +archived -inbox -- tag:inbox and date:..2025-12-31

# Count — useful in shell prompts and dashboards
notmuch count tag:inbox and tag:unread
notmuch count --output=threads tag:flagged

# Reply — emit a header-correct draft (front-end pipes this into $EDITOR)
notmuch reply id:20260504123456.foo@example.org

# Dump / restore — back up the tag database (messages are already on disk)
notmuch dump > tags.dump
notmuch restore < tags.dump

# Compose with shell pipelines
notmuch search --output=files tag:invoice and tag:unread \
  | xargs -I{} mv {} ~/mail/processed/
```

Pair with a sync tool (`mbsync` / `offlineimap` / `getmail` /
`isync`) on a cron / systemd timer; pair with a front-end
(emacs `notmuch.el`, [`alot`], [`astroid`], `notmuch-mutt`,
or [`aerc`](../aerc/) which speaks notmuch as a backend) for
day-to-day reading.

## Why use it

- **Tag-based, not folder-based.** A message can carry any
  number of tags (`inbox`, `unread`, `customer/acme`,
  `project/q3`); search queries combine them with `and` /
  `or` / `not`. Folders are emulated as tags. The mental
  model is Gmail labels, not IMAP folders.
- **Sub-100ms search across millions of messages.** Xapian is
  a mature inverted-index engine; queries against a 2M
  message archive return in tens of milliseconds. By
  contrast IMAP `SEARCH` against the same archive on a
  remote server is seconds-to-minutes.
- **Minimal CLI surface, rich front-end ecosystem.** `notmuch`
  itself is ~7 verbs. The richness lives in `notmuch.el`,
  [`alot`] (Python TUI), [`astroid`] (GTK), `notmuch-mutt`,
  and [`aerc`](../aerc/)'s notmuch backend. Pick the UX
  that suits you; the index is shared.
- **On-disk Maildir, vendor-free.** The mail itself stays in
  one-file-per-message Maildir. `grep -r`, `find`, `rsync`,
  `borg`, `restic`, `git annex` all work on the archive
  with zero special-case handling. Migrate front-ends or
  even back to IMAP at any time without losing data.

## Vs Already Cataloged

- **Vs [`aerc`](../aerc/):** complementary. `aerc` is a TUI
  mail *client* with a notmuch backend among others; pair
  them — `notmuch` indexes and tags, `aerc` provides the
  reading / composing UX. Or use `aerc`'s built-in IMAP
  backend instead and skip `notmuch` entirely; the
  trade-off is that `aerc`-IMAP search is server-bound and
  much slower at scale.
- **Vs [`himalaya`](../himalaya/):** different layer.
  `himalaya` is a CLI mail *client* (talks IMAP/JMAP, sends
  via SMTP). `notmuch` is the local *index* layer. They
  compose: `himalaya` for sync + send, `notmuch` for
  search + tag, any front-end on top.
- **Vs [`mbsync`](https://isync.sourceforge.io/) /
  `offlineimap` / `getmail`:** `notmuch` does *not* sync
  mail. Pair one of those (IMAP → Maildir) with `notmuch
  new` (Maildir → index) on a timer.
- **Vs [`mu`](https://djcb.github.io/mu/) / [`mairix`]:**
  closest peers. `mu` (also Xapian-backed) emphasises an
  emacs-first UX (`mu4e`); `mairix` is text-only and
  produces virtual Maildirs of search results. `notmuch`
  has the broadest front-end ecosystem and the most active
  release cadence.
- **Vs cloud Gmail search:** `notmuch` matches Gmail's
  search experience locally, against your own archive,
  with sub-100ms latency, no egress. The cost is running
  the sync + index pipeline yourself.

## Caveats

- **You must sync mail separately.** `notmuch` operates on
  Maildir on disk; getting mail into that Maildir is a
  separate problem (`mbsync` / `offlineimap` /
  `getmail` / `fdm` / `fetchmail`). Out-of-the-box on
  macOS this is a 30-minute setup the first time.
- **Initial index is slow; incremental is fast.** First
  `notmuch new` on a 500k-message archive can take 20–40
  minutes on an SSD. Subsequent runs after each sync
  process only deltas (sub-second for a few hundred new
  messages).
- **Tag changes do not propagate to IMAP.** `notmuch tag` is
  local-only; bridging tags back to Gmail labels / IMAP
  flags requires `afew` or a custom hook script.
- **No GUI; no built-in composer.** Reading and composing
  live in your front-end of choice. `notmuch reply` emits
  a header-correct skeleton, but the editor + send step is
  yours to wire up (typically `msmtp` or `sendmail`).
- **GPL-3.0+.** Distributing patched `notmuch` requires
  re-licensing under GPL-3.0+; using it as a personal tool
  has no licensing impact.
