# dsq

> **Run SQL queries against JSON, CSV, Excel, Parquet, ORC, Avro,
> YAML, TSV, log files and more — without loading them into a
> database first.** A single Go binary that wraps an embedded
> SQLite engine and a pluggable file-format reader so
> `dsq users.json 'SELECT name FROM {} WHERE age > 30'` Just
> Works. Pinned to **v0.23.0**
> ([LICENSE](https://github.com/multiprocessio/dsq/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/multiprocessio/dsq>

## TL;DR

`dsq` is the answer to "I have a 200 MB JSON dump from a vendor
API and I want to `JOIN` it against a CSV from the analytics team
and a Parquet snapshot from the data lake — and I do not want to
write a Python script, I do not want to spin up DuckDB / ClickHouse,
and I do not want to figure out which `jq` / `csvkit` / `miller`
incantation does a three-way join." It detects file type from the
extension (`.json` / `.ndjson` / `.csv` / `.tsv` / `.parquet` /
`.orc` / `.avro` / `.xlsx` / `.yaml` / `.logfmt` / Apache common
log / nginx access log), reads each file into a temporary SQLite
table, and exposes them as `{}` (single-file shorthand) or
`{N}` (positional) inside a SQL query. JSON arrays of objects
become rows directly; nested objects flatten into dotted column
names (`address.city`); arbitrary JSONPath-like extraction via
`-s "users[*].profile"` for non-tabular inputs. Output is JSON
by default, `--pretty` for a TTY-friendly column table, `-c` to
emit CSV, and the underlying SQLite means full SQL — `WITH`
CTEs, window functions, `JSON_EXTRACT()`, subqueries, math, all
of it. The `--cache` flag persists the converted SQLite database
so repeated queries against the same source file skip re-parsing
(critical for the "explore a 1 GB Parquet interactively" loop).

## Install

```bash
# Homebrew (macOS / Linux)
brew install dsq

# Direct binary (Linux / macOS / Windows)
curl -L https://github.com/multiprocessio/dsq/releases/download/v0.23.0/dsq-darwin-arm64-0.23.0.zip \
    -o /tmp/dsq.zip && unzip /tmp/dsq.zip -d /usr/local/bin/

# Go (build from source — needs Go 1.21+ and CGO for SQLite)
go install github.com/multiprocessio/dsq@v0.23.0

# verify
dsq --version
```

## Example usage

```bash
# simplest: query one JSON file
dsq users.json "SELECT name, email FROM {} WHERE active = true"

# pretty TTY output
dsq --pretty users.json "SELECT COUNT(*), department FROM {} GROUP BY department"

# join JSON against CSV against Parquet (positional refs)
dsq users.json orders.csv products.parquet '
  SELECT u.name, SUM(p.price * o.qty) AS total
  FROM {0} u
  JOIN {1} o ON o.user_id = u.id
  JOIN {2} p ON p.sku = o.sku
  GROUP BY u.name
  ORDER BY total DESC
  LIMIT 10
'

# nested JSON — flatten on read with --schema
dsq nested.json "SELECT user.profile.city, COUNT(*) FROM {} GROUP BY 1"

# pipe input — type forced via --stdin
curl -s api.example.com/data | dsq --stdin json "SELECT * FROM {} LIMIT 5"

# nginx access log analysis (built-in log format detector)
dsq access.log "
  SELECT status, COUNT(*) AS hits
  FROM {} WHERE request LIKE '/api/%'
  GROUP BY status ORDER BY hits DESC
"

# cache the converted SQLite for repeated exploration of a big
# Parquet file
dsq --cache big.parquet "SELECT COUNT(*) FROM {}"
dsq --cache big.parquet "SELECT region, AVG(latency) FROM {} GROUP BY 1"
# ^^ second query is ~100× faster — no Parquet reparse

# emit results as CSV for downstream tooling
dsq -c orders.json "SELECT * FROM {} WHERE status = 'pending'" > pending.csv
```

## When to choose vs alternatives

Pick **dsq** over [`jq`](../jq/) / [`yq`](../yq/) /
[`miller`](../miller/) / [`gron`](../gron/) when the operation is
SQL-shaped (joins, group-by, window functions, set operations) —
those tools are best for path-extraction and per-record
transformation but get awkward past a single-table aggregation.
Pick **jq** instead for "extract this nested path from a stream
of JSON" or any per-record transformation; pick **miller** for
heavy CSV/TSV column-engineering pipelines without SQL syntax.
Pick over [`q`](https://harelba.github.io/q/) (the Python
`q-text-as-data`) when you want broader format support
(Parquet/ORC/Avro/Excel/YAML/log formats vs q's CSV/TSV-only)
and a single static binary instead of a Python install. Pick
over [`textql`](https://github.com/dinedal/textql) when the
input might be JSON or Parquet, not just CSV (textql is
CSV/TSV-only and unmaintained since 2020). Pick over **DuckDB**
when the workflow is "a one-liner from the shell against
whatever file is in the directory" — DuckDB's CLI is more
powerful (it *is* a real columnar database) but expects you to
write `read_json_auto('users.json')` style table functions and
manage a session; dsq's pitch is "zero-config SQL against any
file, period." Pick **DuckDB** instead when the dataset is large
enough that columnar execution matters (>1 GB Parquet, complex
aggregations) or when you need persistent tables, indexes, and
extensions. Pick over [`octosql`](https://github.com/cube2222/octosql)
for broader format support and an embedded SQLite (full SQL
dialect) vs OctoSQL's streaming-first custom executor; OctoSQL
wins for true streaming queries over unbounded inputs. Pairs
with [`xsv`](../xsv/) / [`qsv`](../qsv/) (preprocess CSV first
if it needs cleanup), with [`fx`](../fx/) /
[`visidata`](../visidata/) (interactive exploration after dsq
narrows the data down), and with LLM-CLI workflows as the "give
the model a tabular slice of this dump" preprocessor.

## Caveats

- **Maintenance status**: the parent project (multiprocessio /
  datastation) wound down active development around 2023; dsq
  itself still works but bug fixes have slowed. For a
  more-actively-maintained equivalent, DuckDB's CLI with
  `read_json_auto` / `read_csv_auto` / `read_parquet` is the
  closest substitute (different ergonomics, more powerful
  engine).
- **In-memory by default**: dsq loads the source file into a
  temporary SQLite database before querying, so a 10 GB JSON
  file needs ~10 GB of disk for the temp DB. Use `--cache` to
  persist it under `~/.dsq-cache/` and reuse across runs.
- **JSON array root assumption**: by default dsq expects the
  top-level JSON value to be an array of objects (one row per
  object). For other shapes use `-s '<jsonpath>'` to point at
  the array, or use `--stdin json` with `jq` doing the
  pre-extraction.
- **CGO required**: the embedded SQLite needs CGO at build time,
  so cross-compiling from Linux to macOS without a CGO toolchain
  fails. Use the prebuilt release binaries.
