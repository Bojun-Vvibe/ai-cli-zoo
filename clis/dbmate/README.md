# dbmate

> **A lightweight, framework-agnostic database migration tool** —
> one Go binary that manages timestamped, idempotent SQL
> migrations against Postgres, MySQL, SQLite, and ClickHouse from
> a single `DATABASE_URL` env var, with no opinion about your
> ORM, your language, or your deployment shape, so the same
> `dbmate up` runs in a Rails repo, a Go service, a Python
> notebook stack, and a CI job without dragging in
> Sequelize / Alembic / Flyway / Liquibase. Pinned to **v2.32.0**
> (commit `2036c594de2cf49e357d14561665f03b7df108c7`,
> [LICENSE](https://github.com/amacneil/dbmate/blob/main/LICENSE), MIT).

Source: <https://github.com/amacneil/dbmate>

## TL;DR

`dbmate` is the schema-migration runner for the polyglot service:
plain `.sql` files in `db/migrations/`, named
`YYYYMMDDHHMMSS_description.sql`, each containing two
fence-delimited sections (`-- migrate:up` and `-- migrate:down`),
applied transactionally in order against whatever engine
`DATABASE_URL` points at. `dbmate new add_users_table` scaffolds
the file, `dbmate up` applies pending migrations and writes a
checksum + timestamp into a `schema_migrations` table, `dbmate
down` rolls back the most recent, `dbmate dump` regenerates a
`db/schema.sql` snapshot of the current schema for code review.
There is no DSL, no Ruby / Python runtime requirement, no
ORM coupling — the migration files are SQL you can paste into
`psql`, and the binary is one ~10 MB Go binary you `COPY` into a
Docker image. Engines: Postgres / MySQL / SQLite / ClickHouse;
auth and TLS come from the standard URL parameters each driver
already understands.

## Install

```bash
# Homebrew
brew install dbmate

# static binary (linux / macOS / windows / arm64) from GitHub Releases
curl -fsSL -o /usr/local/bin/dbmate \
  https://github.com/amacneil/dbmate/releases/download/v2.32.0/dbmate-$(uname -s | tr A-Z a-z)-amd64
chmod +x /usr/local/bin/dbmate

# Docker (great for CI)
docker run --rm -it --network=host \
  -v "$(pwd)/db:/db" \
  -e DATABASE_URL=postgres://app:app@localhost:5432/app?sslmode=disable \
  ghcr.io/amacneil/dbmate up

# Go install
go install github.com/amacneil/dbmate/v2@v2.32.0

# verify
dbmate --version    # 2.32.0
```

Requires only the `DATABASE_URL` env var (e.g.
`postgres://user:pass@host:5432/db?sslmode=require`,
`mysql://user:pass@host/db`,
`sqlite:./local.db`,
`clickhouse://user:pass@host/db`) — no config file unless you want one
(`.dbmate.yml` or flags override defaults like `--migrations-dir db/migrations`,
`--schema-file db/schema.sql`).

## One Concrete Example

```bash
# 1. point at a database
export DATABASE_URL=postgres://app:app@localhost:5432/app?sslmode=disable
dbmate create               # creates the database if missing

# 2. scaffold a migration
dbmate new add_users_table
# wrote: db/migrations/20260427181500_add_users_table.sql

cat > db/migrations/20260427181500_add_users_table.sql <<'SQL'
-- migrate:up
create table users (
  id         bigserial primary key,
  email      text not null unique,
  created_at timestamptz not null default now()
);
create index users_email_idx on users (lower(email));

-- migrate:down
drop table users;
SQL

# 3. apply
dbmate up
# Applying: 20260427181500_add_users_table.sql
# Writing: db/schema.sql

# 4. status / rollback / re-apply
dbmate status               # lists applied + pending
dbmate down                 # rolls back the last migration
dbmate up                   # re-applies it

# 5. CI shape (one-liner that fails the build on a missing migration)
DATABASE_URL=postgres://... dbmate --no-dump-schema up
```

`dbmate dump` regenerates `db/schema.sql` from the live database
(via `pg_dump --schema-only` etc.) so reviewers see the *full*
post-migration schema in the PR diff, not just the delta.

## Niche It Fills

**The `migrate`-style runner for teams that don't want a
migration tool to be a programming-language commitment.** Rails'
`ActiveRecord::Migration`, Django's `manage.py migrate`, Sequelize,
TypeORM, Alembic, and Knex all bundle migrations into the
application's framework — fine when the app *is* the framework,
painful when you want migrations to run from a CI image that has
none of the app's runtime, or when multiple services share one
database. `dbmate` is the framework-agnostic answer: plain SQL
files, one binary, one URL.

## Vs Already Cataloged

- **Vs [`goose`](../goose/):** `goose` (pressly) is the closest
  peer — also Go, also `.sql` migrations with `-- +goose Up` /
  `-- +goose Down` fences, also one binary. `dbmate` keeps a
  smaller engine matrix (PG / MySQL / SQLite / ClickHouse) and
  ships a `dbmate dump` snapshot workflow as a first-class verb;
  `goose` adds Go-function migrations, more engines, and an
  embeddable library. Pick `dbmate` for the simpler "SQL files +
  CLI only" posture; pick `goose` if you want to embed the
  migrator into a Go binary or write migrations as Go funcs.
- **Vs [`sqlfluff`](../sqlfluff/):** `sqlfluff` lints / formats
  the SQL inside a migration file; `dbmate` runs it. Pair
  them in a pre-commit hook —
  `sqlfluff lint db/migrations/ && dbmate status`.
- **Vs [`duckdb`](../duckdb/) / [`sqlite-utils`](../sqlite-utils/):**
  those are query / one-shot-mutate tools for a single engine.
  `dbmate` manages an *ordered, versioned, reversible* migration
  history — what you reach for when "the table is already in
  prod" is the constraint.
- **Vs [`harlequin`](../harlequin/) / [`usql`](../usql/):**
  interactive REPLs for ad-hoc SQL. `dbmate` is the
  non-interactive deploy-shaped counterpart — write the
  migration in your editor, run it from CI, rollback with
  `dbmate down` if smoke fails.

## Caveats

- Engine matrix is intentionally narrow (Postgres / MySQL /
  SQLite / ClickHouse) — no SQL Server, no Oracle, no Snowflake,
  no BigQuery. If your warehouse is Snowflake, this is the wrong
  tool; if your prod store is one of the four, it is the right
  tool.
- Migrations are append-only timestamps — there is *no* squash
  / merge / rebase command. Long-lived branches that both add
  migrations will conflict on `db/schema.sql` (which is
  regenerated, so the conflict is mechanical) but not on the
  migration files themselves (different timestamps coexist).
  Adopt a "rebase before merge" rule and re-run `dbmate up`
  locally to refresh `schema.sql`.
- Rollbacks are best-effort — a `down` that does
  `drop column` on Postgres is destructive, and a `down` that
  fails halfway leaves the database in a partial state.
  Production posture is "roll forward with a new migration,"
  not "trust `dbmate down` against prod."
- `dbmate dump` shells out to the engine's native dumper
  (`pg_dump`, `mysqldump`, `sqlite3 .schema`,
  `clickhouse-client`); those binaries must be on `$PATH` of
  whichever host runs the dump (CI image, dev laptop). The
  migration runner itself does *not* need them — only the
  dump step.
- Transactional DDL is engine-dependent — Postgres wraps each
  migration in a transaction by default (good); MySQL silently
  commits between DDL statements (so a half-applied migration
  is a real failure mode). Author MySQL migrations as one
  atomic-able statement where possible, or split into multiple
  files.
