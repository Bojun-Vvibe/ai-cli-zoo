# miniflux

> **Minimalist self-hosted feed reader** — a single static Go
> binary (no runtime deps, no Node, no PHP, no ORM) that
> polls Atom 0.3/1.0, RSS 1.0/2.0, and JSON Feed 1.0/1.1
> sources into Postgres, exposes a distraction-free web UI plus
> a Fever / Google Reader API surface, and ships a complete CLI
> (`miniflux -refresh-feeds`, `miniflux -info`,
> `miniflux -reset-password`, `miniflux -export-user-feeds`)
> for cron / systemd / Ansible workflows — pinned to **v2.2.19**
> (commit
> [`26d9195`](https://github.com/miniflux/v2/commit/26d9195d210410e9d3523316952f9a13ee5f4b1a),
> [LICENSE](https://github.com/miniflux/v2/blob/2.2.19/LICENSE),
> Apache-2.0).

Source: <https://github.com/miniflux/v2>

## TL;DR

The "I want my own feed reader" problem usually splits into
three rough buckets:

1. **A local-only TUI** — [`newsboat`](https://newsbeuter.org/)
   on a laptop, opening links in `$BROWSER`. Good until you
   have two devices or want to read on a phone.
2. **A heavyweight self-hosted reader** — Tiny Tiny RSS,
   FreshRSS, FreshRSS-derived FreshAPI deployments, etc. PHP +
   MySQL/Postgres + a full Apache/Nginx setup, themable but
   slow to upgrade, and the JS surface is *large*.
3. **A SaaS reader** — Feedly, Inoreader, NewsBlur. Solves the
   "two devices" problem at the cost of: their tracking, their
   pixel-stripping policy, their pricing tier, their downtime.

`miniflux` is the right shape for the gap in the middle: a
single self-contained Go binary (~25 MB, statically linked, no
CGO), Postgres as the only hard dependency, a web UI that
intentionally has no JS framework (vanilla JS, ~50 KB total),
and a privacy posture that is *the product* — pixel trackers
stripped on ingest, `utm_*` / `fbclid` / etc. stripped from
every URL, FeedBurner redirects unwound to the canonical link,
external JS blocked by CSP, a built-in media proxy so HTTPS
pages don't mixed-content-warn on HTTP `<img>` tags, and
YouTube embeds rewritten to `youtube-nocookie.com`.

## Install

```bash
# Single binary (Debian / Ubuntu / RPM packages provided)
# Example: download the v2.2.19 .deb for amd64
curl -LO https://github.com/miniflux/v2/releases/download/2.2.19/miniflux_2.2.19_amd64.deb
sudo dpkg -i miniflux_2.2.19_amd64.deb

# Or via Docker (multi-arch incl. ARM64 + RISC-V)
docker pull miniflux/miniflux:2.2.19

# Or build / install from source (requires Go 1.22+)
go install -ldflags="-X 'miniflux.app/v2/internal/version.Version=2.2.19'" \
  miniflux.app/v2@v2.2.19

# Verify
miniflux -version          # → miniflux 2.2.19
miniflux -info             # → build metadata, go version, etc.
```

The Docker images are published to Docker Hub
(`miniflux/miniflux`), GHCR (`ghcr.io/miniflux/miniflux`), and
Quay (`quay.io/miniflux/miniflux`); pin by tag, not by `latest`,
so you can roll back deterministically.

## Day one

```bash
# 1. Postgres prerequisite (v12+)
sudo -u postgres createuser -P miniflux
sudo -u postgres createdb -O miniflux miniflux
sudo -u postgres psql miniflux -c 'CREATE EXTENSION hstore;'

# 2. Minimal config: /etc/miniflux.conf
sudo tee /etc/miniflux.conf >/dev/null <<'CONF'
DATABASE_URL=postgres://miniflux:CHANGEME@localhost/miniflux?sslmode=disable
LISTEN_ADDR=127.0.0.1:8080
RUN_MIGRATIONS=1
CREATE_ADMIN=1
ADMIN_USERNAME=admin
ADMIN_PASSWORD=CHANGEME
CONF

# 3. One-shot DB migration + admin bootstrap
sudo miniflux -migrate
sudo miniflux -create-admin

# 4. Start the daemon (systemd unit ships in the .deb / .rpm)
sudo systemctl enable --now miniflux

# 5. CLI surface for ops without ever opening the web UI
miniflux -refresh-feeds          # force an out-of-cycle poll
miniflux -reset-password         # interactive
miniflux -export-user-feeds USER # OPML to stdout
miniflux -flush-sessions         # log everyone out
```

OPML import / export, full-text search (Postgres-native, no
Elasticsearch), and a REST API with first-party Go and Python
clients are all in the box. Existing mobile apps that speak
the Fever or Google Reader API (Reeder, Fiery Feeds, FeedMe,
NetNewsWire's GR-API mode) point at `https://your-host/v1/`
and Just Work.

## Why pin to v2.2.19

- **Security**: removes sensitive values (CSRF tokens, OAuth
  state, session cookies) from log messages; switches Google
  Reader API auth from SHA-1 to HMAC-SHA-256; uses
  constant-time comparison for token validation; verifies OIDC
  ID-token signatures and claims; rejects oversized favicons;
  fixes a potential DoS when truncating large untrusted input
  in templates.
- **Performance**: meaningful query reductions on the
  unread-entries and main UI pages; uses `SKIP LOCKED` in
  archive operations so a long archive job no longer blocks
  the polling worker; reduces allocations in sanitizer / media
  proxy / routing / template hot paths; caches keymaps in the
  UI instead of recomputing on every keypress.
- **Operations**: graceful shutdown for the worker pool and
  metrics collector (clean systemd restarts no longer drop
  in-flight polls); fixes a CORS preflight regression (now
  correctly returns 204).

## When NOT to reach for it

- **You don't want to run a database.** Postgres is mandatory
  — there is no SQLite mode, no embedded KV, no "just point at
  a directory." If your threshold is "single binary, single
  file, double-click to run," look at
  [`newsboat`](https://newsbeuter.org/) (local TUI, SQLite) or
  [`yarr`](https://github.com/nkanaev/yarr) (single-binary
  web reader with embedded SQLite).
- **You want plugins / themes / a marketplace.** `miniflux` is
  intentionally non-pluggable. You get a custom CSS hook and a
  custom JS hook on the user profile, and that is the entire
  extension surface. The integration list is hard-coded in the
  Go source.
- **You need a non-Postgres backend** (MySQL, MariaDB, SQLite,
  CockroachDB). Upstream has rejected this repeatedly — the
  project leans on Postgres-specific features (`hstore`,
  full-text search, `SKIP LOCKED`, JSONB) and won't abstract
  them.
- **You want a multi-tenant SaaS-style deployment with
  per-tenant billing / quotas.** Multiple users are supported
  (each with their own feed list and admin can create
  accounts), but there is no quota system, no tenancy isolation
  beyond per-user feed lists, and no billing hooks. Wrap it in
  your own front door if you need that.
- **You want client-side encryption of feed contents at rest.**
  Articles live in plain Postgres rows; if your threat model
  requires E2E, this isn't the tool.

If those exclusions don't apply, `miniflux` is the
self-hosted-reader sweet spot: one binary, one database, a
real CLI, a real REST/Fever/GReader API, and an
intentionally-bounded surface that has not bit-rotted in seven
years.
