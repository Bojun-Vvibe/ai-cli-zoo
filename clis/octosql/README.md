# octosql

## What it does
A **command-line SQL engine that joins across heterogeneous file
formats and live databases in one query**. `octosql 'SELECT ... FROM
sales.csv s JOIN users.json u ON s.user_id = u.id'` parses CSV / JSON
/ Parquet / line-JSON / XLSX directly, and via plugins reaches into
PostgreSQL, MySQL, Redis, ClickHouse, Kafka, MongoDB and more — every
data source is a relation in the same `FROM` clause, with a real
streaming SQL planner (predicate pushdown, projection pruning,
hash-join / stream-join chosen by cardinality) underneath. Schema is
auto-inferred from the file header / first records, plugins install
through `octosql plugin install ...` into `~/.octosql/`, and the
engine ships as a single static Go binary.

## Why it's interesting
Different shape from `duckdb` (in-process columnar OLAP, brilliant on
files but no first-class story for joining a CSV against a live
PostgreSQL table without an extension — and no plugin marketplace),
from `q` / `textql` (CSV-only, no remote sources, no plugin model),
from `trdsql` (similar idea — files + RDBMS via SQL — but smaller
plugin surface and no streaming planner), and from a real warehouse
(`bq` / `snowsql`: requires the data to already be in the warehouse).
octosql is the *one-off federated SELECT across "this CSV on disk",
"that JSON dump", and "the prod read-replica"* shape: pick it
specifically when the query is ad-hoc, the sources are mixed live +
file, and you do not want to ETL into a warehouse first. Do **not**
pick it for steady-state analytical workloads on a single file format
(use `duckdb`), for `jq`-shaped tree transforms (use `jq` or `dasel`),
or for anything that needs window functions / CTEs at the bleeding
edge of standard SQL (the dialect is intentionally a useful subset).

## Niche category
Federated streaming SQL CLI — joins files + live databases in one
query via a plugin model.

## Repo
https://github.com/cube2222/octosql

## Version pinned
`v0.13.0` (latest tagged release per
`gh api /repos/cube2222/octosql/releases/latest`)

## License
- SPDX: `MPL-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install cube2222/octosql/octosql

# Go
go install github.com/cube2222/octosql@latest

# Prebuilt binaries (Linux / macOS / Windows)
# https://github.com/cube2222/octosql/releases/tag/v0.13.0
```

## Usage examples
```sh
# Plain CSV → SQL
octosql "SELECT category, SUM(amount) FROM sales.csv GROUP BY category ORDER BY 2 DESC"

# Join a local JSON file against a CSV, output as JSON
octosql -o json \
  "SELECT u.name, SUM(s.amount) AS total
   FROM sales.csv s JOIN users.json u ON s.user_id = u.id
   GROUP BY u.name HAVING total > 1000"

# Install a plugin and query a live database
octosql plugin install postgres
octosql "SELECT * FROM postgres.public.orders LIMIT 50"

# Join a file against a live database in one query
octosql "SELECT o.id, o.total, c.email
         FROM postgres.public.orders o
         JOIN customers.csv c ON o.customer_id = c.id
         WHERE o.created_at > NOW() - INTERVAL '7 days'"

# Output as a table, CSV, or stream JSON
octosql -o table  "SELECT status, COUNT(*) FROM logs.jsonl GROUP BY status"
octosql -o stream_json "SELECT * FROM events.parquet WHERE level='error'"
```
