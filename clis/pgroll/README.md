# pgroll

> **Zero-downtime, reversible PostgreSQL schema migrations using the
> expand / contract pattern with both old and new schema versions
> live simultaneously.** A Go CLI from Xata that runs a migration as
> a *start → backfill → complete* (or *rollback*) lifecycle so old
> and new application versions can read and write through the same
> database during a deploy. Pinned to **v0.16.1**
> ([LICENSE](https://github.com/xataio/pgroll/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/xataio/pgroll>

## TL;DR

Traditional migration tools (`flyway`, `liquibase`, `goose`,
`golang-migrate`, Rails / Django / Alembic migrations) execute a
schema change as one atomic transaction: at `T0` the app sees the old
schema, at `T0+ε` it sees the new one, and any deployed instance
running the old code is now broken. `pgroll` removes that cliff: a
migration is declared as a JSON or YAML file describing the *intent*
(`add_column`, `rename_table`, `change_column`, `drop_column`,
`set_not_null`, …) and the tool implements each operation as a pair
of Postgres views over the underlying physical table — one view per
schema version. While a migration is `in_progress`, *both* the old
and new schemas are queryable through their respective views, with
triggers keeping the underlying columns in sync. You can deploy the
new app code, drain the old one, then `pgroll complete` to drop the
old view. If anything goes wrong before complete, `pgroll rollback`
returns the database to the pre-migration state without data loss.

## Install

```bash
# Homebrew (macOS / Linux)
brew install xataio/pgroll/pgroll

# Go (any platform with Go >=1.22)
go install github.com/xataio/pgroll@v0.16.1

# Docker
docker pull xata/pgroll:v0.16.1

# verify
pgroll --version       # 0.16.1
```

## License

Apache-2.0 — see
[LICENSE](https://github.com/xataio/pgroll/blob/main/LICENSE).
Permissive, vendor-friendly, safe to embed in commercial deploy
pipelines or wrap in a closed-source SaaS migration runner.

## One Concrete Example

```bash
export PGROLL_PG_URL="postgres://app:app@localhost:5432/appdb"
export PGROLL_SCHEMA="public"

# 0. one-time init: creates pgroll's bookkeeping schema
pgroll init

# 1. write a migration that adds a NOT NULL column with a backfill expression
cat > 20260501_add_status.json <<'JSON'
{
  "name": "20260501_add_status",
  "operations": [
    {
      "add_column": {
        "table": "orders",
        "column": {
          "name": "status",
          "type": "text",
          "nullable": false,
          "default": "'pending'"
        },
        "up": "CASE WHEN shipped_at IS NULL THEN 'pending' ELSE 'shipped' END",
        "down": "NULL"
      }
    }
  ]
}
JSON

# 2. start the migration: creates a new view "public_20260501_add_status"
#    with the new column, backfills via the up expression, leaves the old
#    "public" view intact so currently-deployed app instances keep working
pgroll start 20260501_add_status.json

# 3. deploy app code that reads/writes via the new schema view
#    (search_path=public_20260501_add_status,public)

# 4. once all old instances are drained, finalize
pgroll complete

# --- alternative: something is wrong, return to baseline ---
pgroll rollback   # only valid before complete; drops the new view + column
```

## Niche It Fills

**Lets you ship a destructive Postgres schema change (drop column,
rename table, tighten a constraint, swap a type) without a deploy
freeze and without an N+1 / N / N-1 hand-coded shim layer in the
application.** Every "real" zero-downtime migration tutorial ends up
explaining expand-migrate-contract by hand: add nullable column,
deploy code that double-writes, backfill with a script, deploy code
that reads from the new column, drop the old column, deploy code that
stops writing the old column. `pgroll` collapses that 5-step,
multi-day choreography into `pgroll start` → deploy → `pgroll
complete`.

## Why use it

1. **Old and new app versions coexist on the same database during a
   deploy.** Each migration creates a new Postgres view (e.g.,
   `public_20260501_add_status`) alongside the existing one. Old app
   instances point `search_path=public`; new ones point at the new
   schema view. Triggers on the underlying physical table keep them
   in sync until you run `complete`.
2. **`up` and `down` are SQL expressions, not migration scripts.**
   For an `add_column` op, `up` is the SQL expression that backfills
   the new column from existing data; `down` is the inverse used by
   the *other* view so old reads keep working. There is no separate
   backfill script to run, retry, or checkpoint — `pgroll` does it
   in batches with progress accounting in the bookkeeping schema.
3. **Reversible until `complete`.** A migration in `in_progress`
   state can be `pgroll rollback`-ed at any time, dropping the new
   view and any added columns / triggers. After `complete`, the old
   view is dropped and rollback requires a normal forward migration.
   This makes migration deploys behave like feature flags rather
   than like one-way doors.

## Vs Already Cataloged

- **Vs [`atlas`](https://atlasgo.io) (not cataloged) /
  [`dbmate`](../dbmate/) / [`flyway`](https://flywaydb.org)
  (not cataloged):** classic migration tools execute schema DDL
  transactionally. `pgroll` is a *deployment strategy* on top of
  Postgres, not a generic SQL migration runner — it accepts only
  the operation set it can express as expand/contract views.
  Many teams keep both: `dbmate` / `atlas` for trivial additive
  migrations, `pgroll` for the destructive ones.
- **Vs [`dolt`](../dolt/):** `dolt` is a different database with
  git-shaped versioning. `pgroll` is a CLI for vanilla Postgres
  you already run.
- **Vs ORM-built-in migrations (Rails / Django / Alembic):** those
  emit DDL the same way — one transaction, instantaneous schema
  swap. `pgroll` is what you reach for when "schedule a 4 AM
  maintenance window" is not acceptable.

## Caveats

- **Postgres-only.** No MySQL / SQLite / SQL Server backends, by
  design — the view + trigger machinery is Postgres-specific.
- **Operation set is constrained.** Only operations expressible as
  expand/contract are supported (`add_column`, `drop_column`,
  `rename_table`, `rename_column`, `change_column`, `set_not_null`,
  `add_index`, `drop_index`, `create_table`, `drop_table`, `raw_sql`
  with both up and down). Arbitrary SQL still goes through `raw_sql`
  but loses some safety guarantees.
- **Application must opt in via `search_path`.** Both old and new
  app instances need to set their `search_path` to the appropriate
  versioned schema for their lifetime. Apps that hard-code
  `public.<table>` defeat the dual-view machinery.
- **Bookkeeping lives in your DB.** `pgroll init` creates a
  `pgroll` schema for migration state; back it up with the rest of
  the database.
- **Pre-1.0.** API and migration-file format are stable enough for
  production at Xata and a number of community deployments, but
  expect occasional breaking changes between minor versions until
  v1.0; pin a specific version in CI.
