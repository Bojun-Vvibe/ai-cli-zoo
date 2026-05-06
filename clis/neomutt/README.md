# neomutt

> Snapshot date: 2026-05. Upstream: <https://github.com/neomutt/neomutt>

**A keyboard-driven terminal mail client — the long-running mutt
fork that ships notmuch + native NNTP + sidebar + multi-account in
the upstream binary.**
NeoMutt is a single C binary that reads, threads, composes, sends,
encrypts, signs, and searches mail in your terminal — IMAP / POP3 /
local maildir / mbox / mh / SMTP / sendmail-pipe — with an `~/.muttrc`
config file that controls keybindings, colour schemes, threading
order, header pickling, GPG / S/MIME, sidebar layout, NNTP groups,
and notmuch-backed full-text search across millions of messages.

## Repo + version + license

- Repo: <https://github.com/neomutt/neomutt>
- Latest release: **`20260504`** (2026-05-04)
- License: **GPL-2.0** —
  <https://github.com/neomutt/neomutt/blob/main/LICENSE.md>
- License path in repo: `LICENSE.md`
- Default branch: `main`
- Language: C

## Install

```bash
# Distros all package it
brew install neomutt
sudo apt install neomutt
sudo pacman -S neomutt
sudo dnf install neomutt
sudo apk add neomutt

# Run with default ~/.muttrc (or ~/.config/neomutt/neomuttrc)
neomutt

# Read a single mbox file directly
neomutt -f ~/mail/archive.mbox

# Read a maildir (offlineimap / mbsync sync target)
neomutt -f ~/Maildir/INBOX

# Compose from the command line — pipe a body in
echo "see attached" | neomutt -s "report" -a report.pdf -- alice@example.org

# List active mailboxes (sidebar candidates)
neomutt -Q sidebar_visible

# Headless: dump the configured value of any variable
neomutt -Q index_format -Q sort -Q sidebar_format
```

## Niche

The "**keyboard-driven mail client for people who want their mail
on disk, not in a browser tab**" slot.

Browser-based mail (Gmail, Fastmail, Proton, Outlook on the web)
solves the easy 95% — but if you have ten years of project mail,
six aliases, two work accounts, GPG-signed releases, mailing-list
subscriptions, NNTP newsgroups, and a "search across everything
instantly" requirement that doesn't trust a vendor's search box,
the path is: sync mail to a local maildir with `mbsync` /
`offlineimap`, index it with `notmuch`, and open it with NeoMutt.
The whole stack runs offline, the storage format is one file per
message, and the keyboard surface is consistent across SSH /
tmux / a fresh terminal.

NeoMutt is the actively-developed mutt fork: it ships features
(notmuch backend, NNTP, sidebar, multi-account compose, header
cache, fuzzy completion, encrypted password storage hooks) that
upstream mutt accumulated as third-party patches over a decade
and that the NeoMutt project mainlined.

Useful for:

- **Long-term mail archives** with full-text search via
  `notmuch` — `<F8>` opens a query prompt that searches subject
  + body + attachments across millions of messages in
  milliseconds.
- **Multi-account workflows** — folder-hooks switch your `From:`,
  signature, GPG key, and SMTP relay based on which mailbox the
  current message lives in.
- **GPG-heavy mailing lists** — built-in PGP/MIME and S/MIME
  signing / encryption with key selection driven by `~/.gnupg`,
  supports inline-PGP for legacy lists.
- **NNTP newsgroups** — native NNTP backend (no separate
  `slrn` / `tin` install) for project mailing lists that still
  bridge to gmane / news.gmane.io / Usenet.
- **SSH-only / disk-on-laptop / "no JavaScript in my mail
  client"** scenarios — the entire UI is text, no rendering
  engine, no remote-image leak vectors.

## Why it matters

- **Sidebar in upstream** — NeoMutt's sidebar (folder list with
  unread counts) is mainlined; on plain mutt you needed the
  sidebar patch. `:sidebar_visible = yes` and you have a Mail.app
  / Outlook-shaped folder column.
- **Notmuch backend** — `set virtual_spoolfile = yes` plus
  `virtual-mailboxes "Inbox" "tag:inbox and not tag:archived"`
  defines saved-search folders that act like real mailboxes;
  `<F8>` opens an ad-hoc notmuch query that re-renders the
  index from results.
- **Header cache** — `set header_cache = ~/.cache/mutt` keeps
  per-mailbox header indexes on disk, so reopening a 100k-message
  archive is instant instead of re-pulling from IMAP.
- **Per-folder hooks** — `folder-hook ~/Mail/work
  'set from = "alice@work.example.org"; set signature =
  ~/.signature.work; set sendmail = "/usr/bin/msmtp -a work"'`
  swaps your entire identity context as you change folders.
- **Multi-account compose** — `set sendmail` and `$smtp_url`
  per-account, and `<esc>f` switches the From header on the
  fly while composing.
- **Pluggable storage** — IMAP, POP3, NNTP, maildir, mbox, MH,
  notmuch — and the same key bindings work across all of them
  (so an offline `~/Maildir` triaged in flight feels identical
  to a live IMAP folder on landing).
- **Configurable everything** — `~/.muttrc` controls index
  format, threading order, colour scheme per regex against
  headers / body, keymaps per pane, attachment handlers
  (`auto_view text/html` to pipe HTML mail through `w3m`/`lynx`
  for inline rendering); the wiki has battle-tested example
  configs.
- **Active in 2026** — calendar-versioned releases roughly
  monthly; `20260504` (2026-05-04) is the most recent at
  snapshot time, with detailed changelogs per release.
- **Honest scope** — NeoMutt does not sync mail (use `mbsync` /
  `offlineimap` / `getmail`), does not deliver outbound
  (delegates to `msmtp` / `sendmail` / direct SMTP), does not
  index for full-text (delegates to `notmuch`), and is not a
  calendar / contacts client (pair with `khard` for contacts,
  `khal` / `vdirsyncer` for calendar). The whole point is
  pluggability — every layer is replaceable.
- **GPL-2.0** — pure end-user mail client, license affects
  redistribution of the binary; using it to read your own mail
  has no licensing surface at all.
