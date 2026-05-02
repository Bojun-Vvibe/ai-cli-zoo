# pop

> **Send emails from your terminal.**
> A small Go CLI from the Charm ecosystem that wraps SMTP into a
> single command — compose a message in `$EDITOR` (or pipe one
> in), set To/Cc/Bcc/Subject/Attachments, and `pop` delivers it
> via your provider's SMTP relay. Pinned to **v0.2.1**
> ([LICENSE](https://github.com/charmbracelet/pop/blob/main/LICENSE),
> MIT).

Source: <https://github.com/charmbracelet/pop>

## TL;DR

`pop` is the "send a one-off email from a script" tool. You set
four environment variables (`SMTP_HOST`, `SMTP_PORT`,
`SMTP_USERNAME`, `SMTP_PASSWORD`), then either:

- run `pop` with no arguments to get a Bubble Tea TUI (To, From,
  Subject, Body, Attachments, Send), or
- pass everything on the command line / stdin for headless use:
  `echo "build green" | pop -t alerts@example.com -s "CI green"`.

That's the whole tool. No mailbox, no IMAP, no threading, no
address book — just *send*. The TUI is the polished part; the
headless mode is the part you'll actually use in cron jobs and
CI pipelines.

## Install

```bash
# Homebrew (macOS / Linux)
brew install charmbracelet/tap/pop

# Go
go install github.com/charmbracelet/pop@latest

# Nix
nix-shell -p pop

# Arch (AUR)
yay -S pop-bin

# verify
pop --version
```

Configure once via env (typically in `~/.zshrc` or a sourced
secrets file):

```bash
export SMTP_HOST=smtp.fastmail.com
export SMTP_PORT=465
export SMTP_USERNAME=you@example.com
export SMTP_PASSWORD='app-specific-password'   # never your real one
export PUBLIC_KEY=you@example.com              # default From:
```

## License

MIT — see
[LICENSE](https://github.com/charmbracelet/pop/blob/main/LICENSE).
Permissive, embed-friendly, no CLA gate. Charm's standard terms;
safe to bundle inside a container image used by CI.

## One Concrete Example

```bash
# 1. Interactive TUI: type the message, attach files, send
pop
# Tab between fields; Ctrl-S to send. Useful when you want
# composition with line wrapping and a real cursor.

# 2. One-shot from stdin — the cron / CI form
echo "Nightly backup completed: 14.2 GB, 0 errors." | \
    pop -t ops@example.com -s "[backup] success $(date +%F)"

# 3. With an attachment (e.g. ship a log file from CI)
pop -t qa@example.com \
    -s "Failing test trace from build #4711" \
    -a build-4711.log <<<"See attached for the panic stack."

# 4. Multiple recipients, with Cc and Bcc
pop -t a@x.com,b@x.com \
    -c lead@x.com -b audit@x.com \
    -s "Release notes v3.4.0" \
    -a notes.md <notes.md

# 5. As the notification step in a script
./run-tests.sh
if [ $? -ne 0 ]; then
    tail -n 200 test.log | pop \
        -t me@example.com \
        -s "FAIL: tests on $(hostname) at $(date +%T)"
fi

# 6. From a Makefile target (no shell quoting hell)
release: build
	git cliff --tag v$(VERSION) | \
	    pop -t announce@example.com -s "Release v$(VERSION)"
```

## Niche It Fills

**The "send one email, no MUA setup" gap.** Every "send mail
from a shell" path is awkward: `mail`/`mailx` requires a working
local MTA (and most modern systems don't have one); `sendmail`
ditto; Python's `smtplib` needs a wrapper script every time;
`curl --mail-from` works but the syntax is brutal and no one
remembers it. `pop` is the *single, memorable command* that
wraps SMTP-over-TLS authentication and MIME assembly with the
exact UX a script author wants: subject as a flag, body as
stdin, attachments as `-a`, exit 0 on success.

For an LLM-CLI workflow, `pop` is the **"page a human" output
sink** — when an agent finishes an overnight task and needs to
push a digest somewhere the user actually checks (their inbox)
without standing up a webhook or Slack app. Two lines of bash:
one heredoc, one `pop -t you -s …`.

## Why use it

Three things `pop` does that `curl`/`mailx` don't:

1. **TLS-by-default to a real SMTP relay.** Implicit TLS on 465
   and STARTTLS on 587 both work; no need to remember which
   `--ssl-reqd` / `--mail-rcpt` flags `curl` wants this week.
2. **First-class attachments.** `-a file.log -a report.pdf`
   builds the multipart MIME for you. Doing the same with
   `curl` requires hand-writing MIME boundaries.
3. **Same binary for TUI and headless.** Compose interactively
   when you're at a keyboard, pipe stdin in CI, identical
   delivery code path. No "one tool for ad-hoc, another for
   automation".

## Vs Already Cataloged

- **Vs [`himalaya`](../himalaya/):** `himalaya` is a *full email
  client* — IMAP folders, threading, search, send, multiple
  accounts, configuration in TOML. `pop` is just "send", no
  reading, no folders, no state. Use `himalaya` when you want a
  CLI inbox; use `pop` when you only ever need outbound from
  scripts.
- **Vs [`aerc`](../aerc/):** `aerc` is an interactive TUI mail
  client (read, reply, manage). `pop` is non-interactive
  outbound-only. Different layers entirely.
- **Vs `curl --mail-rcpt`:** Same wire protocol underneath, but
  every option name is different and you have to assemble MIME
  yourself for attachments. `pop` is the "I'd rather not look
  this up again" wrapper.
- **Vs an SMTP API client (Mailgun / Postmark / SES SDK):**
  Provider SDKs are correct for high-volume transactional mail
  with templating, suppression lists, and webhooks. `pop` is
  for low-volume "ping me when X happens" — the kind of mail
  that doesn't deserve a service account.

## Caveats

- **Outbound only.** No IMAP, no POP3 (despite the name — the
  "pop" is Charm's, not the protocol's), no folder navigation,
  no replies-as-thread. If you need to *read* mail in the
  terminal, see [`himalaya`](../himalaya/) or
  [`aerc`](../aerc/).
- **Credentials in env.** `SMTP_PASSWORD` lives in your shell
  environment. For shared boxes, source it from a file that's
  `chmod 600` and owned by you — or, better, use a per-host
  secret manager (`pass`, `gopass`, 1Password CLI) and inject
  at call time.
- **Provider quirks bite.** Gmail requires an app password (or
  OAuth, which `pop` does not implement); Office 365 requires
  modern auth in many tenants and will silently reject SMTP
  basic auth — verify with your provider before wiring `pop`
  into a critical alert path.
- **No retry / queue.** A transient SMTP failure exits non-zero
  and the message is lost. For alerting, wrap calls in your
  own retry loop or send through a relay (Postfix in
  submission mode) that handles queueing.
- **HTML is your problem.** `pop` sends `text/plain` by default.
  HTML body support exists but you assemble the markup
  yourself; for most "ship me a tail of the log" use cases,
  plain text is the right answer anyway.
