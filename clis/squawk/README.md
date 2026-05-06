# squawk

> **squawk** — sbdchd/squawk, a linter for PostgreSQL migrations and
> raw SQL that catches the schema-change footguns (locks held too
> long, columns added with defaults that rewrite the table, indexes
> built without `CONCURRENTLY`, constraints validated without
> `NOT VALID`, etc.) **before** the migration ships to a production
> database. Pinned to **v2.49.0**, Apache-2.0 — license file:
> [LICENSE-APACHE](https://github.com/sbdchd/squawk/blob/master/LICENSE-APACHE)
> (also dual-licensed MIT via `LICENSE-MIT`).

Source: <https://github.com/sbdchd/squawk>

## TL;DR

`squawk path/to/migration.sql` parses each statement with a Postgres
grammar and runs it through ~30 named rules each describing a
specific class of operational hazard:

- `adding-required-field` — adding a `NOT NULL` column without a
  default rewrites the whole table under an `ACCESS EXCLUSIVE` lock
- `adding-not-nullable-field` / `adding-field-with-default` —
  variations on the same write-amplification trap on Postgres
  versions where the rewrite still applies
- `disallowed-unique-constraint` — `ALTER TABLE ... ADD CONSTRAINT
  ... UNIQUE (col)` blocks writes; the safe shape is
  `CREATE UNIQUE INDEX CONCURRENTLY ...` then `ADD CONSTRAINT ...
  USING INDEX ...`
- `prefer-robust-stmts` — wrap each statement in its own transaction
  and use `IF NOT EXISTS` / `IF EXISTS` so reruns on partial failure
  don't blow up
- `prefer-text-field` — `VARCHAR(n)` is no faster than `TEXT` and
  changing the limit is a rewrite; use `TEXT` + a `CHECK` constraint
- `require-concurrent-index-creation` — `CREATE INDEX` (without
  `CONCURRENTLY`) holds an `ACCESS EXCLUSIVE` lock for the duration
- `transaction-nesting` — squawk knows `CREATE INDEX CONCURRENTLY`
  cannot run inside a transaction block, so a tool that wraps every
  migration in `BEGIN/COMMIT` will silently break it
- ~20 more covering renames, type changes, foreign keys, and so on

The output is one finding per line with `file:line` + rule code
(SQUAWK rule codes map 1:1 to flag names), parseable by `reviewdog`,
GitHub Actions problem matchers, and most editor LSP integrations.
The exit code is non-zero on any rule violation so a single
`squawk migrations/*.sql` step in CI is the production gate.

It also speaks **Postgres CST natively** (uses `libpg_query`), not a
regex hack — so a multi-statement file with comments, dollar-quoted
function bodies, `DO $$ ... $$` blocks, and partitioned-table DDL
parses correctly where naive linters fall over.

## Install

```bash
# Single static Rust binary — releases at
# https://github.com/sbdchd/squawk/releases/tag/v2.49.0

# npm wrapper (downloads the right release binary on postinstall)
npm install -g squawk-cli

# Homebrew
brew install squawk

# Cargo
cargo install squawk-cli --version 2.49.0

# Docker
docker pull sbdchd/squawk:v2.49.0
```

## Example commands

```bash
# Lint a single migration
squawk migrations/0042_add_user_index.sql

# Lint every migration in CI, fail on any violation
squawk migrations/*.sql

# Disable a specific rule (per file or globally)
squawk --exclude prefer-text-field migrations/*.sql

# Per-rule severity in a config file
cat > .squawk.toml <<'EOF'
[lint]
excluded_rules = ["prefer-text-field"]

[lint.rules.require-concurrent-index-creation]
level = "warning"
EOF
squawk --config .squawk.toml migrations/*.sql

# JSON output for piping into reviewdog / problem matchers
squawk --reporter json migrations/*.sql

# GitHub PR comment integration via the official action
# https://github.com/sbdchd/squawk-action
```

## Niche

PostgreSQL migration linter — applies operational-safety rules to
schema-change SQL before it reaches a production database.

## Why it matters

The expensive failure mode of database migrations is not "the SQL is
wrong" (the type checker catches that) but "the SQL is correct and
also locks the table for 40 minutes during peak traffic". Generic
SQL formatters (`pg_format`, `sqlfluff`) check style, not
operational hazard. ORMs that generate the migrations (`alembic`,
`prisma`, `django`, `ecto`) emit the lock-heavy form by default
because that is the simplest correct DDL — they are not the layer
that should know about your traffic patterns.

Squawk is the rule engine for the operational layer: every rule
encodes a specific class of incident a human operator has previously
filed a postmortem about. Wiring it into PR review (via
`squawk-action`) means the engineer adding the column gets the
warning before the on-call gets paged.

Pick over [`sqlfluff`](https://github.com/sqlfluff/sqlfluff) when
the verb is *operational safety* not *style + dialect compliance*
(sqlfluff is a multi-dialect SQL formatter / style checker; squawk
is Postgres-specific operational lint — they compose: sqlfluff
formats the migration, squawk checks it won't lock the table). Pick
over hand-written CI grep-checks (`grep -i 'create index' | grep -v
concurrently`) for the maintained rule pack. Pair with
[`pgroll`](../pgroll/) (pgroll is the *expand-and-contract* migration
runner that handles the multi-step zero-downtime dance; squawk is
the linter that catches the single-step shape that needs to become
a pgroll workflow), [`pgcli`](../pgcli/) (interactive shell to
explore schema after migration), and [`atlas`](../atlas/)
(declarative-schema migration planner).

Caveats — Postgres-only (no MySQL / SQLite / MSSQL support; the
parser is `libpg_query`, the rules encode Postgres lock semantics);
some rules (`prefer-text-field`) are stylistic and worth excluding
if your team standard is `VARCHAR(n)`; the rule pack reflects
PostgreSQL operational behavior at a snapshot in time, so newer
Postgres versions sometimes loosen a rule (e.g. PG11+ made many
`ADD COLUMN ... DEFAULT` cases metadata-only) and squawk's defaults
err on the safe side — read each rule's documentation before
disabling it for "false positives".

## Verified facts

- Repo: <https://github.com/sbdchd/squawk>
- Latest release tag: `v2.49.0`
- License: Apache-2.0 (also MIT) — `LICENSE-APACHE` at repo root
- Language: Rust (with `libpg_query` for parsing)
