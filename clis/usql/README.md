# usql

> **Universal command-line interface for SQL databases** — one Go
> binary that speaks `psql`-style REPL ergonomics (`\d`, `\dt`,
> `\copy`, `\watch`, `\timing`, `\?`) against ~50 different database
> engines through their native drivers, so the same muscle memory
> you have for Postgres works against MySQL, SQLite, ClickHouse,
> DuckDB, Snowflake, BigQuery, Trino, Cassandra, MongoDB, Redis,
> Oracle, SQL Server, Vertica, CockroachDB, TimescaleDB, Spanner,
> Athena, and more — addressed by a single `driver://user:pass@host/db`
> URL. Pinned to **v0.21.4** (commit
> `f7d0fbe808a87e9f6c726e5de2cec1fa284e88f5`,
> [LICENSE](https://github.com/xo/usql/blob/main/LICENSE), MIT).

Source: <https://github.com/xo/usql>

## TL;DR

`usql` is `psql` for everything else. Connect with a URL
(`usql pg://user@host/db`, `usql mysql://...`, `usql duckdb:./local.db`,
`usql sqlite3:./app.db`, `usql snowflake://...`, `usql bq://project/dataset`),
get a REPL with backslash commands ported from `psql`'s
playbook (`\dt` lists tables, `\d table` describes, `\l` lists
databases, `\copy` streams CSV in / out without server-side perms,
`\watch 5` reruns the last query every 5 s, `\g file.csv` writes
results to a file, `\i script.sql` includes a script, `\set`
defines variables, `\if` / `\elif` / `\else` / `\endif` branch).
Output formats are switchable per-session (`\pset format
aligned|unaligned|csv|json|html|markdown|asciidoc|vertical`), so
the same query becomes a Markdown table for a doc, JSON for a
script, or CSV for downstream tooling without leaving the REPL.
The whole thing is one statically-linked Go binary that ships
the drivers for every supported engine baked in (most builds — a
few drivers like Oracle / DB2 require CGO and live in separate
build tags).

## Install

```bash
# Homebrew (macOS / Linux — recommended)
brew install xo/xo/usql

# Go install (drivers compiled per build tag — `most` is the default tag)
go install -tags most github.com/xo/usql@latest

# Docker
docker run --rm -it ghcr.io/xo/usql:latest pg://user@host/db

# Static binaries from GitHub Releases for linux / macOS / windows / freebsd
# https://github.com/xo/usql/releases/tag/v0.21.4

# verify
usql --version    # usql 0.21.4
usql --help
```

For drivers that require CGO (Oracle `oracle://`, DB2 `db2://`,
Informix, Tibero), build with the appropriate Go build tag — the
default `most` tag covers Postgres / MySQL / SQLite3 / SQL Server /
ClickHouse / DuckDB / Snowflake / BigQuery / Trino / Presto /
CockroachDB / TimescaleDB / Vertica / Cassandra / MongoDB / Redis
and ~30 more pure-Go drivers without any compile step.

## One Concrete Example

```bash
# 1. open a Postgres session — psql-equivalent
usql pg://app@db.internal:5432/app
# app=> \dt              -- list tables
# app=> \d users         -- describe a table
# app=> \timing on
# app=> select count(*) from events where created_at > now() - interval '1 day';

# 2. switch to DuckDB without leaving the binary
usql duckdb:./analytics.duckdb
# analytics=> \copy products from products.csv with (format csv, header true)
# analytics=> select category, count(*) from products group by 1 \g report.json
#                                                    -- writes JSON to file

# 3. one-shot from the shell — no REPL, machine-readable output
usql -c "select id, email from users limit 10" \
     -J pg://app@db.internal/app                      # -J = JSON output
# [{"id":1,"email":"alice@example.com"}, ...]

# 4. include a script and parameterise it
usql -v target_date=2026-04-01 \
     -f migrations/snapshot.sql \
     pg://app@db.internal/app
# uses :target_date inside snapshot.sql

# 5. cross-engine — copy a query result from Postgres into SQLite
usql -c "\copy (select * from events where created_at > '2026-04-01') to events.csv with csv header" \
     pg://app@db.internal/app
usql -c "\copy events from events.csv with csv header" \
     sqlite3:./local.db
```

## Niche It Fills

**The `psql`-shaped REPL for the polyglot-database era.** A team
that runs Postgres in prod, DuckDB for analytics notebooks, SQLite
for tests, and Snowflake for the warehouse used to need four
different CLIs (`psql`, `duckdb`, `sqlite3`, `snowsql`) with four
different sets of meta-commands and four different output-format
flags. `usql` collapses all of them into one binary with one
URL-driven connection model and the `psql` backslash-command
vocabulary as the lingua franca, so muscle memory transfers and
shell aliases stop being engine-specific.

## Vs Already Cataloged

- **Vs [`duckdb`](../duckdb/) / [`sqlite-utils`](../sqlite-utils/):**
  those are engine-specific (`duckdb` is the DuckDB shell;
  `sqlite-utils` is the SQLite Swiss-army CLI). `usql` is the
  *cross-engine* REPL — use `duckdb` when you live entirely in
  DuckDB, use `usql` when your day touches Postgres + DuckDB +
  Snowflake in the same hour.
- **Vs [`harlequin`](../harlequin/):** `harlequin` is a Textual
  *TUI* with multi-pane schema tree + multi-buffer editor +
  results grid; `usql` is a one-line REPL. Use `harlequin` for
  exploratory schema browsing, `usql` for the `psql`-style
  one-liner from a script or `tmux` pane that pipes to `jq` or
  redirects to a file.
- **Vs [`sqlfluff`](../sqlfluff/):** `sqlfluff` is a SQL *linter
  / formatter*; `usql` *executes* SQL. Pair them in a hook —
  `sqlfluff lint migrations/` then `usql -f migrations/up.sql`.
- **Vs [`datasette`](../datasette/):** `datasette` is a
  read-only HTTP browser for SQLite; `usql` is a read-write
  shell for ~50 engines.

## Caveats

- "Universal" is the goal, not a guarantee — engine-specific SQL
  dialect features are *not* abstracted (a Postgres `RETURNING`
  clause does not magically work against MySQL). `usql` unifies
  the *interface*, not the SQL.
- A few drivers (Oracle, DB2, Informix, Tibero) require CGO and
  the right native client libraries on the host (`libclntsh` for
  Oracle, etc.); the default `most` build tag skips these. Read
  the `drivers/` README before assuming a driver works in the
  pre-built binary you downloaded.
- Backslash-command coverage is broad but not 100% `psql`-parity —
  some PostgreSQL-only commands (`\df+`, `\sf`, `\ev`) have
  reduced or different behaviour against non-Postgres engines. Run
  `\?` in a session to see what is supported on the current
  connection.
- Connection-string secrets in shell history is the usual
  hazard — prefer environment variables (`PGPASSWORD`, etc.),
  `~/.usqlrc` (which can `\set` variables and `\connect`), or a
  password helper rather than baking creds into the URL.
- Pre-1.0 (current `v0.21.4`) — flags and backslash-command
  semantics still drift between minor releases; pin the version
  in any CI image and review the changelog before bumping.
