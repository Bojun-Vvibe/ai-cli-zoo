# atlas

- **Repository:** https://github.com/ariga/atlas
- **Latest version:** v1.2.0
- **License:** Apache-2.0 — verified at [`LICENSE`](https://github.com/ariga/atlas/blob/master/LICENSE) (raw: https://raw.githubusercontent.com/ariga/atlas/master/LICENSE)
- **Niche:** Database schema-as-code for many engines, with diff / plan / apply lifecycle

## What it does

`atlas` treats a database schema (Postgres, MySQL, MariaDB, SQLite,
SQL Server, ClickHouse, Redshift, TiDB) as a declarative artifact —
HCL, plain SQL, or an introspection of a live DB — and computes the
migration plan to move any source state to any target state.

```
atlas schema inspect -u "postgres://localhost/app?sslmode=disable" > schema.hcl
atlas schema diff --from "file://schema.hcl" --to "postgres://localhost/app"
atlas migrate diff add_users_table --dir "file://migrations" --to "file://schema.hcl" --dev-url "docker://postgres/16/dev"
atlas migrate apply --dir "file://migrations" --url "$DATABASE_URL"
atlas migrate lint --dir "file://migrations" --dev-url "docker://postgres/16/dev" --latest 1
```

## Why interesting

The boring half of the migration story (write the SQL, apply it in
order, record what ran) is solved by a dozen tools. The interesting
half — *given the current schema and the desired schema, what is the
correct migration?* — is what most teams still hand-write and most
ORMs get subtly wrong (silently dropping a column, reordering a
unique constraint, swapping a `text` for a `varchar(255)` because
the introspection collapsed them).

`atlas` is built around that diff. The dev-database step (`--dev-url
docker://postgres/16/dev`) spins an ephemeral engine, replays the
migration, and validates that the resulting schema actually matches
the declared target — which catches the class of bugs where the
migration "applied cleanly" but produced a schema the next migration
no longer makes sense against. `atlas migrate lint` then runs static
checks for destructive operations, missing concurrent-index hints,
and backward-incompatible changes against the *previous* migration,
not just syntax.

The same engine handles both **versioned migrations** (the classic
`migrations/0001_*.sql` ordered list) and **declarative apply**
(`atlas schema apply` reconciles a live DB to an HCL/SQL file
without writing a migration at all), so a project can start
declarative and graduate to versioned without changing tools.

## Pairs well with

- [`pgroll`](../pgroll/) — when the engine is Postgres specifically
  and you need *zero-downtime* expand/contract semantics that
  `atlas migrate apply` does not give you for destructive changes.
- [`dbmate`](../dbmate/) / [`dolt`](../dolt/) — older / orthogonal
  tools in the same neighborhood; `atlas` is the one with the
  richest cross-engine diff engine and the dev-database validation
  step.
- [`sqlfluff`](../sqlfluff/) — for linting the SQL inside the
  migrations themselves before `atlas migrate lint` checks
  *semantic* drift.

## When to skip

- Single-engine, single-environment, hand-written migrations that
  ship through `psql -f` and a Makefile — `atlas` is overkill and
  adds a binary + a dev-DB requirement to your CI.
- You need **zero-downtime** destructive Postgres migrations
  (`DROP COLUMN`, `RENAME`, type tightening) — reach for `pgroll`,
  which does expand/contract via versioned views; `atlas` will
  cheerfully plan the destructive change and let you ship it.
