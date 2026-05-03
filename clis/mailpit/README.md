# mailpit

> **A self-contained email + SMTP testing tool for
> developers — runs an SMTP server, an HTTP API, and a
> web UI in a single static Go binary, captures every
> message your app sends to localhost:1025, and lets you
> inspect headers / raw source / HTML / plaintext / link
> checks / spam score / HTML compatibility from a browser
> at localhost:8025**, with a JSON-over-HTTP API for
> assertion in integration tests. Pinned to **v1.29.7**
> ([LICENSE](https://github.com/axllent/mailpit/blob/develop/LICENSE),
> MIT).

Source: <https://github.com/axllent/mailpit>

## TL;DR

The classic dev-loop problem: your application sends
"forgot password" / receipt / verification emails. You do
not want those leaking to real inboxes from a staging or CI
run, and you also do not want to mock the mail layer so
deeply that you stop testing the actual template / headers /
multipart structure. `mailpit` is the small daemon that sits
in the middle. Point your app's SMTP client at
`localhost:1025`, it accepts every message (no auth, no
TLS required), persists them to an embedded SQLite store,
and exposes them via a web UI and a REST API. The successor
to MailHog (which is now archived); the same capture-SMTP
pattern, rewritten with a faster pipeline, an HTML preview
that simulates ~90 mail clients (Outlook 2019, Gmail web,
iOS Mail, etc.) via Caniemail data, and an API that returns
JSON instead of MailHog's stringly typed payloads.

## Install

```bash
# Homebrew (macOS / Linux)
brew install mailpit

# Pre-built release tarball (macOS arm64)
curl -L https://github.com/axllent/mailpit/releases/download/v1.29.7/mailpit-darwin-arm64.tar.gz \
    | tar xz
sudo install mailpit /usr/local/bin/

# Linux x86_64
curl -L https://github.com/axllent/mailpit/releases/download/v1.29.7/mailpit-linux-amd64.tar.gz \
    | tar xz
sudo install mailpit /usr/local/bin/

# Docker (the most common CI usage)
docker run -d --name mailpit \
    -p 1025:1025 -p 8025:8025 \
    axllent/mailpit:v1.29.7

# Go install (head, not pinned)
go install github.com/axllent/mailpit@v1.29.7

# verify
mailpit version    # v1.29.7
```

`mailpit` listens on `:1025` (SMTP) and `:8025` (HTTP UI +
API) by default and stores messages in an embedded SQLite
database. With no flags it keeps the DB in memory and loses
everything on restart, which is the right default for a
short-lived CI container.

## Use it for

```bash
# Just run it — SMTP on 1025, web UI on http://localhost:8025
mailpit

# Persist to disk so messages survive restart
mailpit --database ./mailpit.db

# Bind to all interfaces (e.g. for a docker-compose dev env)
mailpit --listen 0.0.0.0:8025 --smtp 0.0.0.0:1025

# Cap retention so the DB doesn't grow forever in long-running envs
mailpit --max 5000              # keep last 5000 messages
mailpit --max-age 72h           # auto-delete after 72h

# Forward / relay matching messages out to a real SMTP server
mailpit --smtp-relay-config ./relay.yml
# (relay.yml: rules by recipient regex + upstream SMTP creds)

# Configure the SMTP listener with auth + TLS for staging-like tests
mailpit --smtp-auth-file ./users.txt \
        --smtp-tls-cert ./cert.pem --smtp-tls-key ./key.pem

# Send a test message into mailpit from any SMTP client
swaks --to alice@example.com --from app@example.com \
      --server localhost:1025 \
      --header "Subject: hello" --body "test"

# Query messages via the REST API (the integration-test surface)
curl -s http://localhost:8025/api/v1/messages | jq '.messages[0].Subject'

# Fetch one message's full structure (parts, headers, attachments)
ID=$(curl -s http://localhost:8025/api/v1/messages | jq -r '.messages[0].ID')
curl -s http://localhost:8025/api/v1/message/$ID | jq

# Search by query string (Bleve-backed full-text index)
curl -s "http://localhost:8025/api/v1/search?query=from%3Aapp%40example.com+subject%3Ahello"

# Delete every message (CI step: reset between scenarios)
curl -X DELETE http://localhost:8025/api/v1/messages

# Run the HTML-compatibility check on a captured message
curl -s http://localhost:8025/api/v1/message/$ID/html-check | jq '.Total'
```

The web UI is the demo surface; the JSON API at
`/api/v1/...` is what makes `mailpit` useful in CI. A
typical Playwright / pytest test triggers the app's
"send password reset" flow, polls
`GET /api/v1/search?query=to:user@test` for ~5 seconds, then
asserts on the returned message's subject, body, and link
list (the API can extract anchor URLs and even HEAD-check
each one for you).

## Why include it in a CLI catalog

1. **It is the live successor to MailHog.** MailHog (the
   classic "fake SMTP for devs" tool) was archived in 2024.
   Every CI pipeline that pinned `mailhog/mailhog:latest`
   is now pinned to an unmaintained image. `mailpit` is
   API-compatible enough to be a near-drop-in (different
   default ports, same SMTP-capture model, richer JSON), and
   the upstream is active (a release every ~2 weeks through
   2025–2026). For a catalog that already lists modern
   replacements next to legacy tools, this is the
   replacement entry.
2. **The HTML-compatibility check is rare and useful.**
   `mailpit` ships an embedded copy of the Caniemail
   compatibility database and reports, per captured message,
   which CSS / HTML features will silently break in Outlook
   2019, Gmail mobile, iOS Mail, etc. Most "fake SMTP"
   tools stop at "show me the raw bytes". For a team
   shipping transactional email templates, this turns the
   capture daemon into a lint step.
3. **One static binary, no Java / Ruby / Node / DB.**
   Competitors in this space (smtp4dev, MailCatcher,
   Papercut, MailHog) are .NET / Ruby / .NET / Go
   respectively. `mailpit` is a single Go binary plus
   embedded SQLite — `docker run --rm -p 1025:1025
   axllent/mailpit` is the entire setup. For a CI
   container where boot time matters, this is the smallest
   surface in the niche.

For an LLM-CLI workflow, the `/api/v1/search` endpoint
returns structured JSON (`From`, `To`, `Subject`, `Date`,
`Snippet`, `Tags`) that an agent can poll while running an
end-to-end test, then assert on without having to parse
mbox / EML by hand.

## Vs Already Cataloged

- **Vs [`swaks`](../swaks/):** orthogonal — `swaks` is the
  Swiss-army-knife SMTP *client* (send a crafted message to
  any server, useful for testing your MTA). `mailpit` is the
  *server* side of the same workflow (catch whatever your
  app sends). The two pair: `swaks --server localhost:1025`
  to inject a test message that `mailpit` then displays.
- **Vs [`aerc`](../aerc/) / [`himalaya`](../himalaya/):**
  orthogonal — `aerc` and `himalaya` are real mail user
  agents you live in (read your IMAP inbox). `mailpit` is
  not for you, it is for your application — a sink for SMTP
  output during dev / CI, with no IMAP fetching and no
  account configuration.
- **Vs an SMTP relay (msmtp, postfix, sendmail):**
  orthogonal — those forward mail toward delivery.
  `mailpit` deliberately *does not* deliver: messages stop
  at the capture DB unless you configure
  `--smtp-relay-config`, which is exactly what you want in
  a non-prod environment.

## Caveats

- **Default storage is in-memory.** Without `--database
  path.db`, messages are lost on restart. Fine for CI,
  surprising the first time you `docker restart` a dev
  container and your captured invite link disappears.
- **No authentication on the web UI by default.** The HTTP
  UI on `:8025` is open to anyone who can reach the port.
  Use `--ui-auth-file` (htpasswd-style) before exposing it
  beyond `localhost`, and never expose it on the public
  internet — it contains real captured messages that may
  include password-reset tokens, magic links, etc.
- **Bleve full-text search has size limits.** The default
  search index is fine for ≤100k messages; beyond that the
  per-query latency climbs and `--max` retention is the
  intended escape hatch. Mailpit is a *test* mail sink, not
  an archival store.
- **HTML-check uses a snapshot of Caniemail data.** The
  embedded compatibility data is updated per release; if you
  need bleeding-edge "does this CSS work in Outlook 365 as
  of yesterday" answers, treat the result as a strong hint
  rather than canonical truth and keep
  [caniemail.com](https://www.caniemail.com) bookmarked.
- **Not API-identical to MailHog.** Migrating from MailHog?
  The endpoints are similar (`/api/v1/messages`,
  `/api/v1/search`) but JSON shapes differ — fields are
  PascalCased and `Content` / `Attachments` are nested
  differently. Plan for one round of test-fixture rewrites,
  not a transparent swap.
- **Last release v1.29.7 (2026-04).** Active upstream;
  release cadence has been ~2 weeks for the last year, so
  pinning by tag is the right move in a CI image.
