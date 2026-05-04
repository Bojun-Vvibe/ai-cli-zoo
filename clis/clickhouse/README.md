# clickhouse

> **Columnar OLAP database with a single-binary CLI client + an
> embeddable `clickhouse-local` for ad-hoc analytics over local
> files** — the same statically-linked binary speaks `clickhouse
> server` (full database), `clickhouse client` (interactive SQL
> REPL with autocomplete + multi-line editing + progress bars),
> `clickhouse-local` (run SQL directly against CSV / Parquet / JSON
> / S3 / HTTP / stdin without a server), and `clickhouse-format`
> (SQL pretty-printer). Pinned to **v26.2.17.31-stable** (released
> 2026-05-04,
> [LICENSE](https://github.com/ClickHouse/ClickHouse/blob/v26.2.17.31-stable/LICENSE),
> Apache-2.0).

Source: <https://github.com/ClickHouse/ClickHouse>

## TL;DR

The "I have a 50 GB Parquet file (or 200 GB of NDJSON, or a
directory of compressed CSVs) and I want to run real SQL
against it without standing up a server" problem has a small
field of good answers — `duckdb`, `datasette`, `qsv`, Polars
in a Python REPL — and `clickhouse-local` is the one that
plays best with **already-existing ClickHouse SQL**, **the
ClickHouse function library**, and **streaming over remote
URLs / S3 / HDFS without download**. The same binary is the
production database engine; the same SQL dialect runs in both
modes. Typical flow: `clickhouse local --query "SELECT count(),
avg(latency_ms) FROM file('logs/*.parquet') WHERE status >=
500"` chews through tens of GB of Parquet at columnar-engine
speed, no server, no schema declaration, no Python. For
LLM-CLI workflows that involve "summarize this trace dump",
"what are the top-50 token-cost call sites in last week's
agent logs", or "join this CSV to that Parquet", `clickhouse-
local` collapses the answer to one line.

## Install

```bash
# macOS / Linux — official one-liner installer
curl https://clickhouse.com/ | sh
# drops `./clickhouse` in the cwd; symlink it onto PATH
sudo mv ./clickhouse /usr/local/bin/

# Homebrew (macOS / Linux)
brew install --cask clickhouse

# Debian / Ubuntu
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | \
  sudo gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] \
  https://packages.clickhouse.com/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update
sudo apt-get install clickhouse-server clickhouse-client

# Verify
clickhouse --version          # ClickHouse 26.2.17.31
clickhouse local --version
```

The single binary multiplexes via the first arg: `clickhouse
server`, `clickhouse client`, `clickhouse local`, `clickhouse
format`, `clickhouse keeper`, `clickhouse benchmark`. Symlinks
named `clickhouse-local`, `clickhouse-client`, etc. dispatch
directly.

## License

Apache-2.0 — see
[LICENSE](https://github.com/ClickHouse/ClickHouse/blob/v26.2.17.31-stable/LICENSE).
Permissive, patent grant included; redistribute and embed
freely. The trademark policy on the "ClickHouse" name is
separate (do not name a fork "ClickHouse"); the code itself
has no field-of-use restriction.

## Common invocations

```bash
# Interactive client against a local server
clickhouse client                 # connects to localhost:9000
clickhouse client -h prod.db -u admin --password --query "SELECT 1"

# Ad-hoc SQL over local files — no server required
clickhouse local --query "
  SELECT toStartOfHour(ts) AS hr, count(), quantile(0.99)(latency_ms)
  FROM file('logs/2026-05-*.ndjson', 'JSONEachRow')
  WHERE status >= 500
  GROUP BY hr ORDER BY hr"

# Stream from S3 directly — no download
clickhouse local --query "
  SELECT count(), uniqExact(user_id)
  FROM s3('https://bucket.s3.amazonaws.com/events/*.parquet',
          'AKIA...', 'secret', 'Parquet')"

# Pipe NDJSON in, SQL out
cat trace.ndjson | clickhouse local --input-format JSONEachRow \
  --query "SELECT span_name, avg(duration_ms) FROM table GROUP BY 1"

# Convert formats — Parquet → CSV → JSON → ORC, all in one binary
clickhouse local --query "SELECT * FROM file('in.parquet') FORMAT CSV" > out.csv
clickhouse local --query "SELECT * FROM file('in.csv') FORMAT Parquet" > out.parquet

# Pretty-print arbitrary SQL
echo "select a,sum(b) from t where c=1 group by a order by 1" \
  | clickhouse format --hilite

# Quick benchmark of a query against the local server
clickhouse benchmark --concurrency 8 --iterations 1000 \
  --query "SELECT count() FROM hits WHERE EventDate = today()"
```

## Why use it

- **One binary, two product surfaces.** The same statically-
  linked artifact is "production OLAP database" *and* "ad-hoc
  Parquet/CSV/NDJSON SQL tool". Local prototyping with
  `clickhouse local` and production `clickhouse server` share
  function library, SQL dialect, and storage format — promote
  a query from notebook to production without rewriting it.
- **Format-aware ingestion is first-class.** `file('*.parquet')`,
  `file('*.ndjson', 'JSONEachRow')`, `s3(...)`, `url(...)`,
  `hdfs(...)`, `mysql(...)`, `postgresql(...)`, `mongodb(...)`,
  `kafka(...)` are all *table functions* you call directly in
  `FROM`. No CREATE TABLE step for ad-hoc analysis. ~70 input
  formats, ~70 output formats, schema inference baked in.
- **Production-shape OLAP, not just a CLI tool.** Vectorised
  execution, primary-key sparse index, MergeTree storage
  family (Replicated, Replacing, Aggregating, Summing,
  Collapsing, VersionedCollapsing), materialised views,
  projections, parallel replicas. The same engine that runs
  Cloudflare's analytics warehouse runs `clickhouse local` on
  your laptop.

## Vs Already Cataloged

- **Vs [`duckdb`](../duckdb/):** the closest peer. `duckdb` is
  a single-binary embeddable OLAP engine with great Parquet /
  Arrow / Pandas integration and a delightful Python REPL.
  Pick `duckdb` when the workload is "analytics in a Python
  notebook" or "I want SQLite-shape embedding inside an app";
  pick `clickhouse-local` when (a) the data lives in S3 / HTTP
  / Kafka / a remote MySQL and you want to query it without
  download, (b) you want the *same* SQL to later run on a
  production ClickHouse cluster, or (c) you need ClickHouse-
  specific functions (`uniqHLL12`, `quantileTDigest`, the
  array DSL, `arrayJoin`, window-frame variants) that DuckDB
  does not have.
- **Vs [`datasette`](../datasette/):** orthogonal. `datasette`
  is "publish a SQLite database as a queryable read-only web
  service with auto-generated UI". `clickhouse` is the
  underlying OLAP engine + CLI. Pair: stage data with
  `clickhouse local --output-format SQLite` (or via
  `INSERT INTO FUNCTION sqlite(...)`), serve with `datasette`.
- **Vs [`qsv`](../qsv/):** `qsv` is a Rust CSV swiss-army knife
  (slice / search / stats / join / index, fast on a single
  machine). `clickhouse local` is SQL-first and reads
  Parquet / NDJSON / S3 in addition to CSV. Use `qsv` for
  "I need to clean and reshape one CSV". Use `clickhouse
  local` for "I need to GROUP BY across 200 files in S3".
- **Vs [`pgcli`](../pgcli/) / [`mycli`](../mycli/) /
  [`litecli`](../litecli/):** those are interactive REPLs for
  specific RDBMSes. `clickhouse client` is the equivalent for
  ClickHouse — autocomplete, syntax highlighting, multi-line
  editing, progress bars, query cancellation with `Ctrl+C` —
  but is bundled in the same binary as the engine itself.

## Caveats

- **Not a transactional OLTP store.** ClickHouse optimises for
  bulk analytic queries on append-mostly data. Single-row
  `INSERT` (one statement, one row) works but is an anti-
  pattern at scale — batch into ≥1000-row inserts, or use
  the `Buffer` table engine. Updates and deletes exist but go
  through async mutations; not designed for "edit this row
  by primary key thousands of times per second".
- **Memory usage scales with query shape.** GROUP BY with
  high cardinality, JOIN with a large right side, ORDER BY
  on unindexed columns all materialise intermediate sets in
  RAM by default. Large analytic queries on a laptop can OOM;
  use `max_memory_usage`, `max_bytes_before_external_group_by`,
  and `max_bytes_before_external_sort` to spill to disk.
- **`clickhouse local` reads files but does not watch them.**
  For "stream from a growing log file" you need `clickhouse
  server` + a `File` engine table or a `Kafka` engine table.
- **Single binary is large.** ~600 MB statically linked
  (server + client + local + keeper + tools all in one). For
  containerised deployments where binary size matters,
  consider stripping to a smaller image or using the official
  `clickhouse/clickhouse-server` image which separates server
  from client builds.
- **SQL dialect is ClickHouse-flavored.** Mostly ANSI-shaped
  but with ClickHouse-specific extensions (`PREWHERE`,
  `SAMPLE`, `ARRAY JOIN`, `LIMIT BY`, materialised columns,
  `ALIAS` columns, the `Nullable` modifier, the
  `LowCardinality` modifier). Queries written here will not
  copy-paste into Postgres or MySQL without translation.
