# prql

> Snapshot date: 2026-05. Upstream: <https://github.com/PRQL/prql>

**A pipelined relational query language that compiles to SQL,
shaped like a `|`-chain so each transform is a single line you
can read top-to-bottom.**
PRQL ("Pipelined Relational Query Language") is a small DSL whose
single purpose is to *transpile* into SQL — Postgres, MySQL, SQLite,
DuckDB, Snowflake, BigQuery, MS SQL, ClickHouse — so you write
`from invoices | filter date > @2024-01-01 | group customer_id
(aggregate {total = sum amount}) | sort -total | take 10` instead
of nested SELECT-from-SELECT-with-CTE-with-window-function SQL,
and the `prqlc` CLI gives you `prqlc compile`, `prqlc watch`, and
a REPL that emits the dialect-specific SQL string ready to paste
into psql / a BI tool / a dbt model.

## Repo + version + license

- Repo: <https://github.com/PRQL/prql>
- Latest release: **`0.13.12`** (2026-04-27)
- License: **Apache-2.0** —
  <https://github.com/PRQL/prql/blob/main/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `main`
- Language: Rust (compiler) + bindings for Python / JS / R / Java / .NET / PHP / Elixir

## Install

```bash
# CLI (single Rust binary)
cargo install prqlc

# Homebrew
brew install prqlc

# Compile a PRQL string to SQL on stdout
echo 'from employees | filter dept == "eng" | take 5' | prqlc compile

# Compile a file, target a specific dialect
prqlc compile -t sql.duckdb query.prql
prqlc compile -t sql.postgres query.prql
prqlc compile -t sql.bigquery query.prql

# Watch mode — recompile on file change
prqlc watch query.prql

# REPL
prqlc

# Inside a dbt model: jinja-include the compiled SQL,
# or use the dbt-prql adapter to keep .prql sources directly.
```

## Niche

The "**`|`-pipelined query language that compiles to vendor SQL**"
slot.

SQL is the lingua franca of analytical databases, but the
`SELECT … FROM … WHERE … GROUP BY … HAVING …` clause order is
the opposite of how queries are *thought* (start with the table,
then filter, then aggregate, then sort, then limit). Window
functions, CTEs, subqueries and dialect quirks compound the
problem: a query that's clear at 5 lines is unmaintainable at
50, and porting from Postgres to BigQuery means rewriting the
date-arithmetic and array functions by hand.

PRQL takes the same approach as `dplyr` (R) and `LINQ` (.NET):
make queries a *pipeline* of small, composable transforms with
the data flowing left-to-right, then transpile to the target
SQL dialect at compile time. Source files live in your repo,
get reviewed in PRs, and produce dialect-specific SQL as a
build artifact — so the warehouse never knows PRQL exists, and
you can switch warehouses without rewriting the analytics layer.

Useful for:

- **Analytics teams that bounce between warehouses** (Postgres
  for staging, BigQuery for prod, DuckDB for local notebooks) —
  one `.prql` source compiles to all three dialects.
- **Long, gnarly aggregation pipelines** with multiple
  group-bys / windows — the linear pipeline stays readable
  where the equivalent SQL becomes a CTE forest.
- **`dbt` models** via the dbt-prql adapter or jinja
  pre-compilation — `.prql` files commit alongside `.sql`
  models, with the same lineage tooling.
- **Ad-hoc CLI exploration** with DuckDB —
  `prqlc compile q.prql | duckdb -c "$(cat -)"` for "query
  this Parquet file" without writing five-level-nested SELECTs.

## Why it matters

- **Pipeline shape, not clause shape** — every transform
  (`filter`, `derive`, `aggregate`, `group`, `sort`, `take`,
  `join`, `window`) is a single line, applied in source order,
  so you read top-to-bottom and the data moves left-to-right
  through the `|` chain.
- **Variables and functions** — `let high_value = filter
  amount > 1000` and `func discount price -> price * 0.9` give
  you actual abstraction over query fragments instead of
  copy-pasting WHERE clauses across reports.
- **Dialect-aware compilation** — `-t sql.postgres`,
  `sql.duckdb`, `sql.bigquery`, `sql.snowflake`,
  `sql.clickhouse`, `sql.mssql`, `sql.sqlite`, `sql.mysql`,
  `sql.glaredb`, `sql.ansi` (default) — date arithmetic and
  function names get translated where dialects diverge.
- **`prqlc watch` for fast iteration** — saves of `query.prql`
  emit the SQL output continuously to a paired file, so a
  side-by-side `nvim query.prql query.sql` gives you a live
  preview of what the warehouse will actually run.
- **Compiles to SQL strings, not a runtime** — your warehouse
  driver, connection pool, and observability stack are
  untouched; `prqlc` is a pure source-to-source transpiler
  invoked at build time or in your editor, not a query
  executor.
- **Bindings for every language that already does SQL strings**
  — `pyprql`, `prql-js`, `prql-java`, `prql.NET`, `prql-elixir`,
  `prql-php`, `prql-r`, so an app that builds SQL strings can
  build PRQL strings instead and call `compile()` at the edge.
- **Active in 2026** — `0.13.12` (2026-04-27) is the most
  recent release at snapshot time; the project releases on a
  3–4 week cadence with a documented stability policy
  (pre-1.0, but with explicit deprecation cycles).
- **Honest scope** — PRQL is a *query language*, not an ORM,
  not a migration tool, not a query optimizer. It does not
  introspect schemas, does not type-check column references
  against a live database (the LSP does some of this against
  configured schemas), and does not handle DDL. Pair with
  your existing migration tool (`sqlx`, `goose`, `alembic`,
  `dbmate`) for schema management.
- **Apache-2.0** — permissive; the compiled SQL output is
  obviously yours, and embedding `prqlc` in a commercial
  product (BI tool, IDE plugin) is fine with attribution.
