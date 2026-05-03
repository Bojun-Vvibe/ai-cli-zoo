# pgweb

- **Repo:** https://github.com/sosedoff/pgweb
- **Version:** 0.17.0 (2025-11-22)
- **License:** MIT ([LICENSE](https://github.com/sosedoff/pgweb/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install pgweb` · `go install github.com/sosedoff/pgweb@latest` · download a pre-built binary from the [releases page](https://github.com/sosedoff/pgweb/releases) · `docker run -p 8081:8081 sosedoff/pgweb` for the container form · single static Go binary, no runtime dependencies

## What it does

`pgweb` is a single-binary cross-platform browser-based client for PostgreSQL: you point it at a connection string (`pgweb --url postgres://user:pass@host/db` or `pgweb --bookmark prod-readonly`) and it boots a small embedded HTTP server (default `:8081`) that serves a one-page web UI for that database. The UI gives you the four things a DBA or developer actually opens a GUI for: a tree of schemas / tables / views / functions / sequences in the left pane; a tab per table with the column list, indexes, constraints, foreign keys, row-count, and a paginated row preview (sortable, filterable per column, with inline `WHERE col LIKE` boxes); a free-form SQL query tab with multi-statement support, query history, and result-grid export to CSV / JSON / XML; and an explain-plan view that runs `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` and renders the plan tree. The connection layer accepts any libpq-compatible DSN (`postgres://`, `postgresql://`, `host=... port=... sslmode=require`), reads `~/.pgpass`, honors `PGSSLROOTCERT` / `PGSSLCERT` / `PGSSLKEY`, and supports SSH tunneling out of the box (`--ssh user@bastion --ssh-key ~/.ssh/id_ed25519`) so you can reach a private RDS / Cloud SQL instance through a jumphost without a separate `ssh -L` step. A read-only mode (`--readonly`) hides every DDL / DML form and refuses non-`SELECT` statements, which makes pgweb safe to hand to a support engineer for a prod incident; a sessions mode (`--sessions`) lets multiple users connect to the same pgweb process, each picking their own DB at login time, which is the form the official Docker image exposes for shared on-call use. Output of every query tab is exportable; the query history is persisted to a local SQLite file under `~/.pgweb/history.db` so you can re-run yesterday's diagnostic without retyping it.

## When to pick it / when not to

Pick `pgweb` when you need a *visual* PostgreSQL browser that is one binary, runs anywhere a Go binary runs (laptop, bastion, dev container, k8s sidecar), and exposes its UI as plain HTTP that any browser or `kubectl port-forward` can reach. Concrete cases: you SSH'd into a bastion to look at prod and you do not want to install a heavy desktop client like DBeaver or pgAdmin; you are debugging a Postgres-backed migration on a developer's laptop and a quick visual `\dt` plus column-level filters beats typing `psql` queries; you want to give a non-SQL teammate read-only access to a staging DB for the afternoon (`pgweb --readonly --listen 0.0.0.0 --bookmark staging-ro`); you are pair-debugging a query plan and want the tree-rendered EXPLAIN output rather than psql's text form; you ship a small internal tool and want to bake a DB inspector sidecar into your dev compose file without dragging in a Java/Electron client. Pair with [`pgcli`](../pgcli/) for the keyboard-first `psql`-with-autocomplete TUI when you would rather stay on the command line; pair with [`dbmate`](../dbmate/) or [`golang-migrate`](../golang-migrate/) for the migration layer (pgweb does not run migrations); pair with [`steampipe`](../steampipe/) when the question is "query AWS / GitHub / random APIs as if they were Postgres tables" rather than "browse this Postgres".

Skip `pgweb` when the database is not Postgres — it is Postgres-only by design (no MySQL, no SQLite, no MSSQL); for SQLite use [`harlequin`](../harlequin/) or [`sqlite-utils`](../sqlite-utils/), for MySQL use [`mycli`](../mycli/), for "any SQL via a JDBC-like URL" use [`usql`](../usql/) or [`harlequin`](../harlequin/) again. Skip when the workflow is fully terminal-native and you do not want a browser involved — `pgcli` and `psql` are faster for the muscle-memory case. Skip when you need a managed multi-tenant DB UI with auth, audit logs, query approvals, and per-role row-level masking — that is the Metabase / Hasura / CloudBeaver Team category, not pgweb. Skip when the DB has billions of rows and you intend to scroll the table view; pgweb's row preview pages reasonably but is not designed to be a data-warehouse explorer (use [`duckdb`](../duckdb/) or [`datasette`](../datasette/) on a snapshot for that).

## Vs already cataloged

- **Vs [`pgcli`](../pgcli/):** complementary. `pgcli` is the keyboard-driven `psql` replacement with autocomplete and syntax highlighting; pgweb is the browser-driven schema-tree + grid + EXPLAIN-tree GUI. Most teams that pick one end up running both — pgcli for "I want to type a query right now in this SSH session", pgweb for "I want to click through the schema with someone over Zoom".
- **Vs [`harlequin`](../harlequin/):** harlequin is a TUI SQL IDE that supports DuckDB, Postgres, SQLite, MySQL, Snowflake, and BigQuery from one cross-engine UI; pgweb is Postgres-only and runs in the browser. Pick harlequin when you live in the terminal and want one tool across many engines; pick pgweb when the audience includes non-terminal users and the DB is Postgres.
- **Vs [`gobang`](../gobang/) / [`lazysql`](../lazysql/):** both are TUI multi-engine DB clients. pgweb is the same shape conceptually but renders to a browser and goes deeper on Postgres (EXPLAIN tree, sequences, functions, materialized views, ssh tunnel built in).
- **Vs [`datasette`](../datasette/):** datasette is a SQLite-first read-only data publisher with a rich plugin ecosystem and a public-API stance ("publish this dataset on the web"); pgweb is a Postgres-only operator-facing inspector ("let me poke at this prod DB"). Different audience, different defaults.
- **Vs [`steampipe`](../steampipe/) / [`octosql`](../octosql/):** orthogonal. Those project external APIs / files as virtual SQL sources; pgweb assumes you already have a real Postgres and wants to give you a UI for it.

## Caveats

- **Postgres only.** No MySQL, no SQLite, no MSSQL, no Oracle. Reads any libpq-compatible DSN and supports Postgres 9.x through 17 in practice; some very old (8.x) servers may miss columns in the schema view.
- **The default listen address is `127.0.0.1:8081`.** If you flip to `--listen 0.0.0.0` (or expose the Docker port to the network) you are publishing a DB UI without auth — put a reverse proxy with auth in front, or use `--auth-user` / `--auth-pass` for HTTP Basic, or stay behind `kubectl port-forward` / SSH tunnel.
- **Read-only mode is a UI guard, not a DB-side enforcement.** `--readonly` filters the SQL pgweb itself will issue, but the connected role can still do anything its `GRANT`s allow if someone bypasses the UI. For real prod safety, give pgweb a connection string that uses a Postgres role with only `SELECT` grants.
- **Bookmarks live in a YAML file at `~/.pgweb/bookmarks/*.yml`** and store DSNs in cleartext. Treat that directory like `~/.aws/credentials`. If you want secret-manager-backed bookmarks, wrap pgweb in a small launcher script that fetches the DSN at start time and passes `--url`.
- **No diff / migrate / seed features.** pgweb is an inspector, not a schema-management tool. Pair with [`dbmate`](../dbmate/), [`golang-migrate`](../golang-migrate/), or [`atlas`](../atlas/) for that layer.
- **The query editor is single-user per process by default.** Sessions mode (`--sessions`) supports multiple concurrent users, but each session is in-memory; if the pgweb process restarts, history and bookmarks for sessions reset (the `~/.pgweb/history.db` is single-user).
- MIT ([LICENSE](https://github.com/sosedoff/pgweb/blob/master/LICENSE)) — permissive, safe to embed in internal tooling, dev containers, and Docker images.

## Example invocations

```bash
# Install
brew install pgweb
go install github.com/sosedoff/pgweb@latest

# Connect via DSN, open default browser to the local UI
pgweb --url postgres://app:secret@localhost:5432/appdb

# Connect via libpq env vars (PGHOST / PGUSER / PGPASSWORD / PGDATABASE)
PGHOST=db.internal PGUSER=ro PGDATABASE=warehouse pgweb

# Read-only mode — hand to a teammate for an afternoon
pgweb --readonly --listen 127.0.0.1:8081 --url postgres://ro@stg-db/app

# SSH tunnel to a private RDS through a bastion, no separate ssh -L
pgweb --ssh ec2-user@bastion.example.com \
      --ssh-key ~/.ssh/id_ed25519 \
      --url postgres://app@10.0.4.21:5432/appdb

# Multi-user sessions mode (the Docker image's default)
pgweb --sessions --listen 0.0.0.0:8081

# Docker
docker run --rm -p 8081:8081 \
  -e DATABASE_URL=postgres://app:secret@host.docker.internal/appdb \
  sosedoff/pgweb

# Bookmarks (~/.pgweb/bookmarks/staging.yml) and pick by name
pgweb --bookmark staging
```
