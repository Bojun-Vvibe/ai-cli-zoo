# turso-cli

- **Repo:** https://github.com/tursodatabase/turso-cli
- **Version:** v1.0.21 (2026-04-28)
- **License:** MIT ([LICENSE](https://github.com/tursodatabase/turso-cli/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install tursodatabase/tap/turso` · `curl -sSfL https://get.tur.so/install.sh | bash` · prebuilt binaries on the GitHub release page · binary name is `turso`

## What it does

`turso` is the command-line client for Turso Cloud, a managed
fork of SQLite (libSQL) that exposes per-tenant SQLite databases
as a network service with **edge replicas**, embedded replicas,
and HTTP-shaped wire protocol so any environment that cannot open
a local file (serverless functions, browser, mobile) can still
speak SQL to a real SQLite engine. The CLI handles the full
lifecycle: `turso auth signup`, `turso db create my-app`, get a
connection URL plus rotating auth token, open a `turso db shell`
REPL with readline + history + multi-line editing, dump / restore
schemas, manage replica locations across the global anycast
network, and mint short-lived tokens scoped to a single database
or read-only access.

It also wraps the local `libsql` binary so `turso dev` spins a
local replica that syncs to the cloud primary, useful for offline
dev against the same schema your production talks to.

## When to pick it / when not to

Reach for `turso` when you want **SQLite semantics with cloud
durability and global read-replicas, and you do not want to
operate a Postgres**. It fits agent backends and side-projects
that need a per-user or per-tenant DB (cheap to spin up tens of
thousands of dbs, each with its own auth token), edge functions
on Cloudflare Workers / Vercel / Netlify that benefit from the
HTTP wire protocol, and mobile / desktop apps that want local-first
SQLite with optional sync.

Skip it when you need full Postgres features (rich types, FTS,
PostGIS, materialized views, advisory locks) — `turso` is SQLite
underneath and inherits SQLite's type-affinity quirks. Skip if your
data is single-region and high-write — a vanilla Postgres on a VM
will outperform any read-replica DB on writes. Skip for true
air-gapped use cases; the value prop is the cloud control plane.

## Why it matters in an AI-native workflow

Agent runs generate a lot of small structured state: per-session
chat history, per-tool call traces, per-evaluation result rows.
SQLite is the right shape for that data, but an in-process file
breaks the moment your agent runs in a serverless function or a
browser tab. Turso's HTTP wire protocol means an agent can keep
"one tiny SQLite per session" semantics while running anywhere,
and the cheap-per-database pricing lets a multi-tenant agent
service give every user their own isolated DB without operational
overhead. The CLI's per-DB token minting (`turso db tokens create
my-app --read-only`) is also a clean way to hand a sandboxed agent
read-only access to its own state.

## Example invocations

```bash
# First-run auth (opens browser)
turso auth signup
turso auth login

# Create a database in a region near the user
turso db create agent-state --location lhr

# List dbs and inspect connection details
turso db list
turso db show agent-state --url
turso db tokens create agent-state --expiration 1h

# Open an interactive SQL shell against the cloud DB
turso db shell agent-state

# Add a read replica in another region
turso db replicate agent-state nrt

# Dump and restore for backups / migrations
turso db shell agent-state .dump > backup.sql
turso db shell agent-state < schema.sql

# Run a local libSQL replica that syncs to the cloud primary
turso dev --db-file ./local.db
```

## Alternatives in this catalog

- [`mycli`](../mycli/) — interactive REPL for MySQL with
  schema-aware completion; reach for mycli when the engine
  underneath is MySQL / MariaDB, turso when it is libSQL / SQLite.
- [`atuin`](../atuin/) — also uses SQLite for sync of shell history;
  similar local-first-with-sync philosophy at a smaller scope.
- [`csvlens`](../csvlens/), [`tabiew`](../tabiew/) — TUIs for
  exploring tabular data once you have dumped it out of turso.
