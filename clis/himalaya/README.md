# himalaya

> **A scriptable, MIME-aware email CLI / TUI that speaks IMAP / Maildir /
> Notmuch / JMAP for reading and SMTP / Sendmail for sending, with
> per-account TOML config and OAuth2 + secret-service backends for
> credentials.** A single Rust binary by Pimalaya. Pinned to **v1.2.0**
> ([LICENSE](https://github.com/pimalaya/himalaya/blob/master/LICENSE),
> MIT).

Source: <https://github.com/pimalaya/himalaya>

## TL;DR

`himalaya` is the answer to "I want my email in the terminal but I do
not want to learn `mutt` / `notmuch` / `aerc` and I do not want a
fork-of-a-fork-of-mh circa 1996". One static Rust binary, one
`config.toml` per account, and email becomes a normal subject for
shell scripting: `himalaya envelope list`, `himalaya message read`,
`himalaya message send < draft.eml`, `himalaya attachment download
<id>`. The CLI mode is fully scriptable (JSON output via `-o json`,
stable exit codes), the TUI mode (`himalaya`) is keyboard-driven
three-pane (folders → message list → message body) with vim-style
navigation, and the same binary speaks IMAP, Maildir on disk, JMAP
(Fastmail), and Notmuch — picked per account in the config file. SMTP
or Sendmail for sending. Passwords come from `keyring` (macOS Keychain
/ libsecret / Windows Credential Manager), GPG / `pass`, or
OAuth2 with PKCE for Gmail / Outlook / Fastmail.

## Install

```bash
# Cargo (any platform with Rust >=1.79)
cargo install himalaya --version 1.2.0 --locked

# Homebrew (macOS / Linux)
brew install himalaya

# Arch
pacman -S himalaya

# Nix
nix-env -iA nixpkgs.himalaya

# Direct binary
curl -sLO https://github.com/pimalaya/himalaya/releases/download/v1.2.0/himalaya.aarch64-apple-darwin.tar.gz

# verify
himalaya --version    # 1.2.0
```

## License

MIT — see
[LICENSE](https://github.com/pimalaya/himalaya/blob/master/LICENSE).
Permissive, redistributable, fine to vendor inside another product.
The Pimalaya umbrella also publishes companion CLIs (`pimalaya
contact`, `pimalaya calendar`) under the same license; the core
`email-lib` crate is reusable for building custom email tooling.

## One Concrete Example

```bash
# 1. one-time config (TOML at ~/.config/himalaya/config.toml)
cat > ~/.config/himalaya/config.toml <<'TOML'
[accounts.personal]
default = true
email = "you@example.org"
display-name = "You"
folder.aliases = { inbox = "INBOX", sent = "Sent", trash = "Trash" }

[accounts.personal.backend]
type = "imap"
host = "imap.example.org"
port = 993
encryption = "tls"
login = "you@example.org"
auth = { type = "passwd", cmd = "pass show email/personal" }

[accounts.personal.message.send.backend]
type = "smtp"
host = "smtp.example.org"
port = 465
encryption = "tls"
login = "you@example.org"
auth = { type = "passwd", cmd = "pass show email/personal" }
TOML

# 2. list the 20 most recent envelopes in the inbox
himalaya envelope list --folder INBOX --page-size 20

# 3. read message #4567 (renders multipart/HTML to text via html2text)
himalaya message read 4567

# 4. download all attachments of one message into ~/Downloads/
himalaya attachment download 4567 --folder INBOX

# 5. send a message via stdin (no editor)
himalaya message send <<'EOF'
From: you@example.org
To: friend@example.org
Subject: shell-script send

hi from a script.
EOF

# 6. JSON output for piping into jq / a TUI of your own
himalaya envelope list -o json | jq '.[] | select(.flags | contains(["Seen"]) | not) | .subject'

# 7. interactive TUI mode (three-pane, vim keys)
himalaya     # j/k, Enter to open, c to compose, r to reply, q to quit

# 8. delete (move to Trash) by ID range
himalaya message delete 4500..4520 --folder INBOX
```

## Niche It Fills

**A modern, scriptable, single-binary email client for the shell that
does not require committing to a maildir-on-disk + offlineimap +
notmuch + msmtp + custom-`mutt`-config workflow.** The terminal-email
ecosystem has historically been "assemble five tools and write 200
lines of glue" (`offlineimap` or `mbsync` to fetch into a Maildir,
`notmuch` to index, `mutt` / `neomutt` to render, `msmtp` to send,
`abook` for contacts). `himalaya` collapses fetch + render + send +
search into one Rust binary configured by one TOML file, with the
maildir + notmuch + IMAP + JMAP options all available as backend
choices — opt into the legacy stack only if you want to.

## Why use it

1. **JSON output makes email scriptable like any other Unix data
   source.** `himalaya envelope list -o json` returns a typed array
   of `{id, flags, subject, from, date}`; pipe to `jq`, feed into
   another CLI, write a per-team triage dashboard in 30 lines of
   shell. The TUI is built on top of the same JSON-emitting library
   so behavior cannot drift between `-o json` and the interactive
   surface.
2. **Backend is a config choice, not a recompile.** The same binary
   reads from IMAP today, Maildir + `mbsync` tomorrow, JMAP at
   Fastmail next month — change `[accounts.<name>.backend]` from
   `type = "imap"` to `type = "maildir"` and the rest of the
   workflow is unchanged. Useful for staged migration from a
   provider-IMAP setup to a local-Maildir + `notmuch` setup
   without rewriting your scripts.
3. **OAuth2 with PKCE for Gmail / Outlook / Fastmail in one binary.**
   `auth = { type = "oauth2", method = "xoauth2", ... }` runs the
   PKCE dance, stores the refresh token in the system keyring, and
   silently refreshes — no app password, no Google "less secure
   app", no separate `oauth2-cli` daemon to keep alive.

## Vs Already Cataloged

- **Vs [`mc`](../mc/):** mc is a file manager that incidentally
  speaks `sh://` and `ftp://`; himalaya is purpose-built for IMAP /
  JMAP / Maildir email semantics (folders, flags, attachments, MIME
  parts), not a generic VFS.
- **Vs [`posting`](../posting/):** posting is a TUI for HTTP /
  REST / GraphQL — a Postman replacement. Different protocol, same
  "give me a keyboard-driven TUI in `tmux` instead of an Electron
  app" instinct.
- **Vs [`gh`](../gh/):** `gh` reads PR review comments and issue
  notifications via the GitHub API; himalaya reads the same content
  delivered as actual emails to your inbox. They compose: `gh pr
  list` for the source-of-truth view, `himalaya envelope list
  --folder GitHub` for the historical archive your provider already
  filtered into a label.

## Caveats

- **No built-in fetcher daemon.** Himalaya queries IMAP / JMAP on
  demand; for offline-first workflows pair with `mbsync` /
  `offlineimap` writing to a Maildir and configure the himalaya
  account with `backend.type = "maildir"`.
- **HTML rendering is text-only.** Multipart/HTML is converted via
  `html2text`; CSS / images / inline tables degrade. For graphical
  HTML mail open in a browser via `himalaya message export <id>
  --raw > /tmp/m.eml` and an external viewer.
- **Calendar and contacts are sibling projects, not bundled.**
  Pimalaya publishes `pimalaya-contact` and `pimalaya-calendar`
  separately; himalaya is mail-only.
- **Pre-2.0.** v1 is stable for daily use, but the config schema
  has churned across minor releases (notably the v0 → v1 split of
  `[backend]` into `[backend]` + `[message.send.backend]`); pin
  a himalaya version per machine and re-read the `CHANGELOG.md`
  before upgrading across minor versions.
- **OAuth2 redirect uses `127.0.0.1`.** First-run OAuth requires a
  browser on the same host as the CLI to complete the PKCE
  redirect; for headless servers, run the OAuth flow on a laptop
  and copy the resulting `~/.local/share/himalaya/...` token cache.
