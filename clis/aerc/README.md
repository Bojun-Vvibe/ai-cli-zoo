# aerc

## What it does
A **terminal email client** in Go that handles IMAP, JMAP, Maildir,
notmuch, and mbox accounts in one TUI; speaks SMTP, sendmail, and
`msmtp`-style outgoing; and builds the message list / message view /
compose flow around modal `vi`-style key bindings and tab-completed
`:command` lines. Multi-account is first-class: each account is a tab,
each folder is a sub-tab, threading is server-side or client-side as the
backend supports it. The compose pipeline shells out to `$EDITOR`,
attachments are added with `:attach`, and outgoing patches use a built-in
`git send-email`-shaped flow (`:patch`, `:patch-apply`) that reads and
writes proper `git-format-patch` series — the reason aerc has the
following it does inside kernel-style mailing-list communities. Optional
notmuch backend turns the same UI into a tag-driven query browser. Single
static binary plus a small set of POSIX filter scripts (`html`,
`hldiff`, `colorize`, `calendar`) it pipes message bodies through.

## Why it's interesting
Different shape from webmail (browser, mouse, vendor lock-in), from
[`himalaya`](../himalaya/) (CLI-shaped — one command per action, scripts
cleanly, no persistent UI; aerc is the *interactive* counterpart), from
mutt / neomutt (the long-standing curses standard — stable, immensely
configurable, but C macros + muttrc DSL; aerc trades some of that
configurability for a cleaner key model and built-in JMAP / patch
review), from Thunderbird (GUI, heavy), and from notmuch's own
`notmuch-emacs` (Emacs-shaped). aerc is the *modern terminal email
client with first-class patch review* shape: pick it specifically when
the workload is a high-volume mailing list (LKML, sourcehut lists, qemu-
devel) and `git send-email` round-trips matter, or when you want IMAP +
notmuch + JMAP behind one consistent vi-style UI. Do **not** pick it
for one-shot scripted "send this email from CI" jobs (use
[`himalaya`](../himalaya/), `mailx`, or `msmtp`), for users who want a
GUI, or as a shared multi-user mail server (it is a client only).

## Niche category
Terminal email client — vi-modal multi-account IMAP/JMAP/Maildir/notmuch
TUI with built-in `git send-email` patch review flow.

## Repo
https://github.com/rjarry/aerc
(read-only mirror of the canonical sourcehut repo at
https://git.sr.ht/~rjarry/aerc)

## Version pinned
`0.21.0` (latest tagged release on `master`)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install aerc

# Arch
sudo pacman -S aerc

# Debian / Ubuntu (recent)
sudo apt install aerc

# Fedora
sudo dnf install aerc

# From source (requires go >= 1.23, scdoc, GNU make)
git clone https://git.sr.ht/~rjarry/aerc
cd aerc
gmake
gmake install PREFIX=~/.local
```

## Usage examples
```sh
# First run launches the account-setup wizard, then drops you into the TUI
aerc

# Inside the TUI (vi-modal):
#   j / k             next / previous message
#   Enter             open message
#   c                 compose new
#   r                 reply
#   R                 reply-all
#   D                 delete
#   a                 archive
#   /                 search
#   Tab               next account
#   :pipe -m | git am -3       apply a patch series from the open thread
#   :patch new <name>          start a new patch set
#   :q                quit

# Send logs to a file (handy when debugging IMAP / JMAP)
aerc > /tmp/aerc.log 2>&1

# Headless one-shot mail send via the bundled bin (when configured)
echo "body" | aerc :send-message recipient@example.org "Subject"
```
