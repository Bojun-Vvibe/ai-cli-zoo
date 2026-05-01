# sqlx-cli

> **Compile-time-checked SQL migrations for Rust** — the companion CLI
> to the `sqlx` async, pure-Rust SQL toolkit. Manages versioned
> up/down migration files, creates and drops databases, and (with
> `cargo sqlx prepare`) snapshots the query metadata that lets the
> `sqlx::query!` macro verify your SQL against a real Postgres /
> MySQL / SQLite / MSSQL schema **at `cargo build` time** — even on
> CI machines that have no live database. Pinned to **v0.8.6**
> (commit `bab1b022bd56a64f9a08b46b36b97c5cff19d77e`,
> [LICENSE-APACHE](https://github.com/launchbadge/sqlx/blob/v0.8.6/LICENSE-APACHE)
> / [LICENSE-MIT](https://github.com/launchbadge/sqlx/blob/v0.8.6/LICENSE-MIT),
> dual Apache-2.0 / MIT).

Source: <https://github.com/launchbadge/sqlx/tree/v0.8.6/sqlx-cli>

## TL;DR

If your Rust service talks to SQL through the `sqlx` crate, `sqlx-cli`
is the missing piece that turns "I have a `migrations/` folder full
of timestamped `.sql` files" into a real, reproducible workflow.
`sqlx migrate add <name>` scaffolds a new
`YYYYMMDDHHMMSS_<name>.up.sql` (and matching `.down.sql` if you pass
`-r` for reversible mode); `sqlx migrate run` applies pending
migrations against `$DATABASE_URL` while recording each one in a
`_sqlx_migrations` ledger table with checksum + execution time;
`sqlx migrate revert` walks one step back if the migration was
declared reversible. `sqlx database create|drop|reset` short-circuits
the usual psql/mysqladmin dance (handy in test suites and dev
containers). The headline feature, though, is `cargo sqlx prepare`:
it points the `query!` / `query_as!` macros at a live database,
type-checks every SQL string in your crate, and writes the resulting
metadata into `.sqlx/` so subsequent `cargo build` runs (in CI, in
release pipelines, in `cargo install`) can compile **without** a
database connection — you get the safety of compile-time SQL
verification without a hard dependency on a live DB at build time.
Works against Postgres, MySQL/MariaDB, SQLite, and MSSQL through the
same `--database-url` flag and the same migration files.

## Install

```bash
# from crates.io (recommended; pick the DB drivers you actually need)
cargo install sqlx-cli --version 0.8.6 --no-default-features \
  --features native-tls,postgres,sqlite

# everything (Postgres + MySQL + SQLite + MSSQL, native-tls)
cargo install sqlx-cli --version 0.8.6

# Homebrew
brew install sqlx-cli

# verify
sqlx --version            # sqlx-cli 0.8.6
```

## Examples

```bash
# 1. Bootstrap a Postgres-backed project from zero
export DATABASE_URL=postgres://app:app@localhost:5432/app_dev
sqlx database create
sqlx migrate add -r create_users          # writes migrations/<ts>_create_users.{up,down}.sql
$EDITOR migrations/*_create_users.up.sql  # CREATE TABLE users (...);
sqlx migrate run                          # applies it, records in _sqlx_migrations
sqlx migrate info                         # show applied vs. pending

# 2. CI-friendly offline build: snapshot query metadata, then build with no DB
cargo sqlx prepare -- --tests             # writes .sqlx/ next to Cargo.toml
git add .sqlx && git commit -m 'chore: refresh sqlx query metadata'
SQLX_OFFLINE=true cargo build --release   # CI runs this with no Postgres in sight
```
