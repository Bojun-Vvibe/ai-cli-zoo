# mblaze

> **Suite of ~25 small Unix-style commands for reading,
> searching, composing, and threading email stored in a
> Maildir** — each program does one thing (`mlist` lists
> messages, `mscan` formats a thread index, `mshow`
> renders one message, `mthread` computes JWZ threading,
> `mpick` filters with a tiny expression language) and
> reads/writes plain message paths on stdin/stdout, so you
> compose mail workflows with `xargs`, `awk`, `grep`, and
> shell pipes instead of a monolithic TUI. Pinned to **v0.2**
> (released 2024-12-30,
> [COPYING](https://github.com/leahneukirchen/mblaze/blob/v0.2/COPYING),
> CC0-1.0 / public domain).

Source: <https://github.com/leahneukirchen/mblaze>

## TL;DR

The terminal email space splits into (a) integrated TUIs
that own the loop — `mutt`, `neomutt`, [`aerc`](../aerc/),
[`himalaya`](../himalaya/), [`meli`](../meli/) — every
action happens inside one process, customization means a
config DSL; and (b) tool-suites in the `mh` / `nmh`
tradition where every operation is a separate executable
operating on Maildir paths, and the "UI" is whatever
combination of shell, `vim`, and `less` you wire together.
`mblaze` is the modern entry in category (b): a clean
rewrite of the `mh` model for Maildir (not MH folders), no
state daemon, no index database, no IMAP client of its own
(point it at an existing Maildir maintained by
[`mbsync`](https://isync.sourceforge.io/) /
[`offlineimap`](https://www.offlineimap.org/) /
[`fdm`](https://github.com/nicm/fdm)), and a JWZ-correct
threader (`mthread`) that most TUI clients still get wrong.

## Install

```bash
# macOS (Homebrew)
brew install mblaze

# Debian / Ubuntu (since bookworm / 22.04)
sudo apt-get install -y mblaze

# Arch Linux
sudo pacman -S mblaze

# From source (C99, no deps beyond libc)
git clone https://github.com/leahneukirchen/mblaze.git
cd mblaze && make && sudo make install

# Verify
mlist -V                  # mblaze 0.2
```

## License

CC0-1.0 (public-domain dedication) — see
[COPYING](https://github.com/leahneukirchen/mblaze/blob/v0.2/COPYING).
You may copy, modify, distribute, and use the work for any
purpose without permission or attribution. Two helper files
(`mystrverscmp.c`, `mymemmem.c`) are MIT-licensed and
attributed to Rich Felker — relevant only if you redistribute
modified source.

## Common invocations

```bash
# One-time setup: tell mblaze where the Maildir is
export MAILDIR=~/Mail/inbox

# List + scan + show: the read loop
mlist -s "$MAILDIR"            # all unread, one path per line
mlist -s "$MAILDIR" | mscan    # formatted index (date | from | subj)
mlist -s "$MAILDIR" | mscan | head -1 | awk '{print $1}' | mshow

# Pick the current message into a shell variable, navigate
mseq -f                                    # show the current sequence
mseq .+1 | mshow                           # next message
mseq .-1 | mshow                           # previous

# Threaded view of a folder
mlist ~/Mail/lists/golang | mthread | mscan -f '%c%17d %20f %t %s'

# Search: full-text via grep, structured via mpick
mlist ~/Mail | mpick -t '/from/ ~ "leah@" && /subject/ ~ "patch"'
mlist ~/Mail | xargs grep -l -i 'maildir'    # body search via grep

# Compose + send (pipes to MTA, e.g. msmtp / sendmail)
mcom alice@example.org -s "Re: patch v3" \
  | $EDITOR /dev/stdin               # or pipe straight to msmtp

# Reply with quoted body, threaded headers preserved
mshow -r `mseq .` | mcom -t            # -r prints reply template

# Tag / mark / move using plain mv on Maildir paths
mflag -S `mseq .`                      # mark seen
mrefile `mseq .` ~/Mail/archive        # move to archive folder
```

## Pipeline patterns this enables

- **Email as text streams**: every command emits Maildir
  paths on stdout, so `mlist | mpick -t '...' | xargs
  mflag -S` is "mark all messages from a list as read" — no
  plugin, no scripting language, just shell.
- **Editor-of-choice composition**: `mcom` writes an RFC
  5322 draft to stdout / a temp file; you edit it in `vim`,
  `helix`, `kakoune`, whatever — there is no embedded
  editor to fight.
- **Pairs cleanly with sync daemons**: keep
  [`mbsync`](https://isync.sourceforge.io/) on a 5-minute
  systemd timer, run `notmuch new` and `mblaze` against the
  same Maildir, and you get the [`notmuch`](../notmuch/)
  search engine *plus* the `mh`-style read loop on the same
  store with no conflicts.
- **Trivially scriptable for triage**: a 10-line shell
  script using `mlist`, `mpick`, `mflag`, `mrefile` will
  out-perform any TUI's filter rule for "auto-archive
  GitHub notifications older than 30 days I have already
  read."

## When NOT to use it

- You want a **single integrated TUI** that handles IMAP,
  reading, writing, and contacts in one window — pick
  [`aerc`](../aerc/), [`himalaya`](../himalaya/), or
  [`meli`](../meli/) instead.
- You **don't already have** a sync layer
  (`mbsync` / `offlineimap` / `fdm`) and don't want to set
  one up — `mblaze` does not speak IMAP / POP / JMAP.
- Your team standard is **HTML-rich mail with inline
  images and calendar invites** — `mshow` renders text and
  fires `mailcap` for parts, but you will spend more time
  configuring `mailcap` than reading.

## Why it earns a slot in the zoo

`mblaze` is the rare modern tool that bets entirely on the
Unix-philosophy half of the design space: ~25 binaries,
~10k lines of C, no daemon, no index, no config file
(operates from `~/.mblaze/profile` if present, sane
defaults if not), public-domain license. Every other
terminal mail client in the zoo is a TUI; `mblaze` is the
*toolkit* — the right answer when you want email to be
just another stream you can `grep`, `awk`, and `xargs`
through, indistinguishable from any other text source on
the system.
