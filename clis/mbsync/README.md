# mbsync (isync)

> **Bidirectional IMAP ↔ Maildir synchroniser** — the binary is
> `mbsync`, the upstream project is `isync`. One C binary opens
> IMAP connections to one or more remote accounts and reconciles
> them against a local Maildir tree on disk, tracking per-message
> UIDs and per-flag changes in `.uidvalidity` / `.mbsyncstate`
> sidecar files so a flag toggle, move, or delete on either side
> propagates correctly to the other on the next run. Pinned to
> **v1.5.1** (released 2025-03-11,
> [COPYING](https://sourceforge.net/p/isync/isync/ci/master/tree/COPYING),
> GPL-2.0-or-later with an OpenSSL linking exception).

Source: <https://sourceforge.net/projects/isync/> (upstream
git: `git clone https://git.code.sf.net/p/isync/isync`)

## TL;DR

`mbsync` is the canonical "pull my IMAP mailbox down to disk
*and* push my local changes back up" engine for terminal mail
clients. Drives a Maildir under e.g. `~/.mail/<account>/<box>/`
that [`mutt`](https://gitlab.com/muttmua/mutt) /
[`neomutt`](https://github.com/neomutt/neomutt) /
[`aerc`](https://git.sr.ht/~rjarry/aerc) /
[`notmuch`](https://notmuchmail.org/) read directly, with full
two-way semantics — read/unread, flagged, deleted, moved-to-
folder all reconcile on the next `mbsync -a`. Per-channel rules
in `~/.mbsyncrc` declare which IMAP folders map to which local
boxes (`Patterns "INBOX" "Archive*" "!Junk"`) and which side
wins on conflict (`Sync All`, `Create Both`, `Expunge Both`),
so the same config can be a one-way archive mirror, a two-way
sync, or a download-only newsfeed depending on directives.
Multiple accounts are independent channels in the same
`mbsyncrc`, run sequentially or in parallel with `-a`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install isync

# Linux package managers
# Debian / Ubuntu: apt install isync
# Fedora: dnf install isync
# Arch: pacman -S isync

# Tarball build (any Unix with autoconf + OpenSSL + Cyrus-SASL headers)
curl -LO https://downloads.sourceforge.net/project/isync/isync/1.5.1/isync-1.5.1.tar.gz
tar xzf isync-1.5.1.tar.gz && cd isync-1.5.1
./configure && make && sudo make install
```

## When to choose mbsync

- You read mail in a terminal client (`mutt` / `neomutt` /
  `aerc` / `notmuch`) and want a local Maildir that stays in
  sync with one or more IMAP accounts.
- You want *bidirectional* sync — flag and folder changes made
  in the local client propagate back to the IMAP server.
- You need offline mail access (laptop on a plane, intermittent
  connectivity) with a guarantee that everything reconciles
  cleanly when the link comes back.
- You want the canonical, smallest, most-debugged engine in
  this niche — `mbsync` is what most "build your own terminal
  mail stack" guides standardise on.

## When to pick something else

- You only need one-way *download* (POP-style fetch + local
  delivery to a script) — `getmail` or `fetchmail` is closer
  to the shape of that workflow.
- You want push notifications via IMAP IDLE persistently — pair
  `mbsync` with [`goimapnotify`](https://gitlab.com/shackra/goimapnotify)
  or `imapnotify`; `mbsync` itself is run-on-demand / cron, not
  a long-lived daemon.
- Your client is a GUI mail app (Thunderbird, Apple Mail) — they
  speak IMAP directly and a separate sync engine adds nothing.
- The remote is Exchange / Graph-only — `mbsync` needs IMAP; use
  the vendor's bridge or [`davmail`](https://davmail.sourceforge.net/).

## Caveats

- Configuration is one global `~/.mbsyncrc` with strict block
  syntax (`IMAPAccount` / `IMAPStore` / `MaildirStore` /
  `Channel` / `Group`). The first run is the steep part; once
  it works it tends to keep working.
- Passwords go via `PassCmd "<command>"` — pipe from
  [`pass`](https://www.passwordstore.org/),
  [`gopass`](https://github.com/gopasspw/gopass),
  `security find-generic-password` (macOS Keychain), or
  `secret-tool` (libsecret). Never put plaintext in `mbsyncrc`.
- OAuth2 (Gmail, modern providers) needs an external token
  helper such as [`mailctl`](https://github.com/pdobsan/mailctl)
  or `oauth2ms` — `mbsync` accepts the token via `PassCmd` but
  does not perform the OAuth dance itself.
- `mbsync -a` is sequential by channel; for many accounts run
  channels in parallel via `xargs -P` or systemd template units.
