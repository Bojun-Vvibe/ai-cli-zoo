# sqruff

> **Fast SQL linter and auto-formatter, Rust port of the
> sqlfluff rule set.** Drop-in style enforcement for ten SQL
> dialects (BigQuery, ClickHouse, Databricks, DuckDB, Postgres,
> Redshift, Snowflake, SQLite, Spark, T-SQL) with sub-second
> startup so it can sit in `pre-commit` and `lefthook` hooks
> without becoming the slow link. Pinned to **v0.38.0**
> ([LICENSE](https://github.com/quarylabs/sqruff/blob/main/LICENSE),
> Apache-2.0; version checked via
> `gh api repos/quarylabs/sqruff/releases/latest`).

Source: <https://github.com/quarylabs/sqruff>

## TL;DR

`sqruff` is what happens when you take the sqlfluff rule
catalog (the de-facto standard for SQL style checks: keyword
casing, comma placement, alias requirements, ambiguous joins,
trailing semicolons, JINJA-aware templating) and rewrite the
parser + linter in Rust on top of a hand-written grammar.
Result: ~10-40x faster than the Python original on the same
file, single ~12 MB binary, and a `--fix` mode that rewrites
the file in place. Configuration in `.sqruff` is
TOML-superset-compatible with sqlfluff's `.sqlfluff` for the
overlapping rule IDs, so a team can migrate incrementally.

## Install

```bash
# macOS / Linux via Homebrew
brew install sqruff

# Cargo
cargo install sqruff

# Single-binary release
curl -L https://github.com/quarylabs/sqruff/releases/latest/download/sqruff-linux-x86_64.tar.gz \
  | tar xz && sudo mv sqruff /usr/local/bin/

# Pre-commit
# - repo: https://github.com/quarylabs/sqruff
#   rev: v0.38.0
#   hooks: [{ id: sqruff }]
```

## Example

```bash
# Lint one dialect
sqruff lint --dialect postgres ./models/

# Auto-fix the fixable rules in place
sqruff fix --dialect snowflake warehouse/transforms/*.sql

# Show available rules with one-line descriptions
sqruff rules

# Use as a CI gate (non-zero exit on any violation)
sqruff lint --format github-actions ./sql/
```

## When to use

- Your repo has more than a handful of `.sql` files and you
  want consistent style enforced in CI without adding 5+
  seconds of Python startup per pre-commit run.
- You're in a polyglot warehouse stack (BigQuery + Snowflake +
  DuckDB) and want one tool that knows all three dialects.
- You already use `ruff` for Python and want the same
  "Rust-fast linter that auto-fixes" experience for SQL.

## When NOT to use

- You need deep semantic analysis (catalog-aware column lineage,
  type checking, dead-code elimination across models) — reach
  for `sqlmesh` / `dbt` `defer` / `pgroll` for schema-aware
  work; sqruff is style + grammar only.
- You depend on a sqlfluff rule that hasn't been ported yet —
  the rule catalog is large but not yet 100% parity. Run
  `sqruff rules` first and stay on `sqlfluff` for the gaps.
- You only have one or two SQL files; the value compounds with
  scale.

## Orthogonality vs existing zoo entries

- **vs [`sqlfluff`](../sqlfluff/)** — same rule philosophy,
  different runtime: sqlfluff is the canonical Python
  implementation (broader rule catalog, slower startup);
  sqruff is the Rust port (faster, single binary, narrower
  but converging rule set). Pick sqruff when CI latency
  matters; pick sqlfluff when you depend on a rule sqruff
  hasn't ported yet.
- **vs [`sqlc`](../sqlc/)** — sqlc generates typed
  application code from SQL queries and a schema; it is a
  codegen tool, not a style checker. Both can coexist on the
  same `.sql` files.
- **vs [`sqlfmt`-style tools / `pg_format`]** — those format
  but do not lint (no rule catalog, no CI severity, no
  per-rule disable comments). sqruff covers both axes.
- **vs [`taplo`](../taplo/) / [`yamlfmt`](../yamlfmt/) /
  [`shfmt`](../shfmt/) / [`stylua`](../stylua/)** — same
  "single Rust/Go binary that lints + formats one language"
  pattern, applied to SQL.

## Niche / tags

`sql` · `linter` · `formatter` · `rust` · `pre-commit` ·
`bigquery` · `snowflake` · `duckdb` · `postgres`
