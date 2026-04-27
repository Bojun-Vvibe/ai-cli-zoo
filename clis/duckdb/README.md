# duckdb

> The **in-process analytical SQL engine** distributed as a single
> static CLI: zero-install, zero-config column-store OLAP that reads
> CSV / Parquet / JSON / Arrow / Excel / Iceberg / Delta directly from
> local disk, S3, GCS, Azure Blob, or HTTPS — and answers SQL over
> billions of rows on a laptop in seconds. Pinned to **v1.5.2**
> (commit `0da6e1e84c60c0d5aebd35d7758fcdb0f8bb6eb7`,
> [LICENSE](https://github.com/duckdb/duckdb/blob/main/LICENSE), MIT).

Source: <https://github.com/duckdb/duckdb>

## TL;DR

`duckdb` is "SQLite for analytics": a single ~30 MB binary, no
server, no daemon, no network, with a vectorized columnar query
engine that out-runs Postgres on aggregations by 10–100×.
`duckdb my.db` opens (or creates) a database file and drops you
into a REPL; `duckdb -c "select count(*) from 's3://bucket/*.parquet'"`
runs a one-shot query directly against remote Parquet without
ingesting a single row first; `duckdb -json -c "select * from
read_csv_auto('events.csv') limit 10"` is the streaming-CSV-to-JSON
filter that makes it a Unix-shaped data tool. The killer trick is
**zero-copy reads**: `read_parquet`, `read_csv_auto`,
`read_json_auto`, `read_ndjson` and the `httpfs` / `aws` /
`azure` / `iceberg` / `delta` extensions let SQL run over files in
their native location, with predicate pushdown and projection
pruning, so a 200 GB Parquet dataset on S3 can be filtered to a
1 MB result without ever materializing the rest.

## Install

```bash
# brew (macOS / linux)
brew install duckdb                  # 1.5.2

# direct download (single binary)
curl -fsSL https://install.duckdb.org | sh

# verify
duckdb --version
```

No dependencies. The same binary backs the Python (`pip install
duckdb`), Node, Rust, R, Java, and JDBC clients — they all share
the embedded engine.

## One Concrete Example

A pile of newline-delimited LLM run logs in `runs/*.jsonl`:

```bash
# 1. ad-hoc count grouped by model, no ingest step
duckdb -c "
  select model, count(*) as n, avg(latency_ms) as p_avg
  from read_json_auto('runs/*.jsonl')
  group by model
  order by n desc
"

# 2. join those logs against a CSV of pricing
duckdb -c "
  with r as (select * from read_json_auto('runs/*.jsonl')),
       p as (select * from read_csv_auto('pricing.csv'))
  select r.model, sum(r.input_tokens * p.input_per_1k / 1000) as cost_usd
  from r join p using (model)
  group by r.model
"

# 3. persist the result for downstream tools
duckdb analytics.db -c "
  create table run_costs as
  select * from read_json_auto('runs/*.jsonl');
"

# 4. stream it back out as Parquet for the next stage
duckdb -c "copy (select * from analytics.db.run_costs)
           to 'run_costs.parquet' (format parquet);"
```

## Niche It Fills

**The default analytics substrate for AI / data CLIs in this catalog.**
Several entries here read or write `.duckdb` files directly
([`harlequin`](../harlequin/), [`dlt`](../dlt/),
[`datachain`](../datachain/)) precisely because the engine has no
server to operate, no schema to declare in advance, and no driver
matrix to maintain — the same file opens from every language. For
the operator who needs to slice a multi-GB Parquet / CSV / JSONL
log without spinning up Postgres or paying for BigQuery,
`duckdb` is the answer in one binary; for the engineer building a
local-first analytics pipeline, it is the engine that everything
else composes against.

## Vs Already Cataloged

- **Vs [`sqlite-utils`](../sqlite-utils/):** SQLite is row-store
  (great for OLTP, key-value lookups); DuckDB is column-store
  (great for OLAP, aggregations on wide tables). Use `sqlite-utils`
  for "store this typed CSV and look it up by id"; use `duckdb`
  for "scan a billion rows and group by something".
- **Vs [`harlequin`](../harlequin/):** `harlequin` is the TUI
  *front-end* that opens DuckDB (and seven other engines) for
  interactive exploration; `duckdb` is the engine itself plus a
  basic REPL. Pick `duckdb` for scripts and pipelines, `harlequin`
  for an interactive editing surface.
- **Vs [`datasette`](../datasette/):** `datasette` serves a SQLite
  file over HTTP for read-only sharing with a browser audience;
  `duckdb` is the local engine for the analyst doing the
  computation. They cover opposite ends of the data lifecycle.
- **Vs warehouse CLIs (Snowflake / BigQuery `bq`):** the warehouse
  CLIs talk to a remote multi-tenant cluster you pay per query;
  `duckdb` runs the same SQL against the same Parquet on your
  laptop for free. Use it as the dev / preview tier; use the
  warehouse for the prod tier.

## Caveats

- DuckDB is **embedded, single-process**. There is no concurrency
  story for multiple writers — open the file from one process at
  a time. For team-shared analytics, materialize results to
  Parquet / a warehouse, do not share the `.duckdb` file over NFS.
- The `httpfs` / `aws` / `azure` extensions are **lazy-loaded** on
  first use. `INSTALL httpfs; LOAD httpfs;` is one-shot per
  database file; CI environments that wipe `~/.duckdb/extensions`
  pay the download on every run — pre-bake an image with the
  extensions installed.
- `read_csv_auto` infers types from the first ~20K rows; mixed
  columns get cast to `VARCHAR`. For canonical schemas use
  `read_csv` with explicit `columns = {...}` rather than `_auto`.
- The on-disk format is stable within a major release line but
  not across major versions — `1.x` files are not guaranteed to
  open in `2.x` (use `EXPORT DATABASE` / `IMPORT DATABASE` for
  long-term archives, or just keep the source Parquet).
