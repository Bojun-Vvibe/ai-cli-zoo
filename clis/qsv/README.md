# qsv

- **Repo:** https://github.com/jqnatividad/qsv
- **Version:** 19.1.0 (latest stable, April 2026)
- **License:** Unlicense / MIT dual ([UNLICENSE](https://github.com/jqnatividad/qsv/blob/master/UNLICENSE), [LICENSE-MIT](https://github.com/jqnatividad/qsv/blob/master/LICENSE-MIT))
- **Language:** Rust
- **Install:** `brew install qsv` · `cargo install qsv --locked --features=apply` · `pacman -S qsv` · `winget install qsv` · static binaries on the GitHub release page · binary name is `qsv`

## What it does

`qsv` is the actively-maintained successor to BurntSushi's `xsv`: a
single-binary, streaming, multi-threaded toolkit for slicing,
indexing, joining, validating, and transforming **CSV, TSV, and
delimited tabular files** — including ones too big to fit in RAM.
The command surface is a verb-per-subcommand list with around 50
operations: `headers`, `slice`, `select`, `search` (regex over
columns), `frequency`, `stats` (with mean / stddev / quartiles /
cardinality), `dedup`, `sort`, `sortcheck`, `join` (inner / left /
right / full / cross / anti / semi), `joinp` (in-memory Polars
join, much faster on big files), `partition`, `pivotp`, `transpose`,
`apply` (regex / case / datefmt / geocode / calc), `validate`
(JSON-Schema), `excel` / `to xlsx` (round-trip with spreadsheets),
`fetch` / `fetchpost` (parallel HTTP enrichment with caching and
rate limiting), `geocode`, `luau` (run a Lua 5.4 script per row),
`py` (run Python per row when built with the feature), `sniff`
(infer schema, delimiter, encoding), and `template` (Jinja-style
row → text rendering). Most subcommands stream — memory stays flat
on a 50 GB file — and several can be sped up further by building a
sidecar **`.idx` index** with `qsv index`, after which `slice`,
`count`, `sample`, and `split` become O(1).

## When to pick it / when not to

Pick `qsv` whenever the data is genuinely tabular and the question
is "filter / aggregate / join / clean / validate / enrich". It is
the right tool for ad-hoc analysis of a multi-GB export when
loading it into pandas would OOM your laptop, for
schema-validating a CSV against a JSON Schema before it reaches a
warehouse, for running a regex find/replace across 200M rows with
deterministic output, for joining two large CSVs without spinning
up DuckDB, and for enriching a CSV by hitting an HTTP API once per
row with built-in caching and rate limiting (no Python script
required). The `stats --everything` command is a one-shot data
profiler that beats writing five pandas cells.

Skip it when the data is fundamentally **relational with multiple
tables and you'd run SQL anyway** — reach for [`duckdb`](../duckdb/),
which will out-perform `qsv` on complex multi-join analytics and
gives you a real query planner. Skip it for **interactive
exploration of small files** (under ~100 MB) where a notebook is
genuinely faster to iterate in. Skip it for **non-tabular data** —
JSON arrays of nested objects are better handled by [`jq`](../jq/),
[`jaq`](../jaq/), or [`miller`](../miller/) (which speaks both
CSV and JSON natively). Skip it as a long-term ETL platform — for
recurring pipelines, materialise into Parquet and let DuckDB or a
real warehouse own the workload; `qsv` is the surgical tool, not
the conveyor belt.

## Why it matters in an AI-native workflow

Coding agents are routinely asked to "look at this CSV and tell me
what's wrong" — a request that, naively, means streaming the
whole file into the context window and burning tokens to no real
end. `qsv stats --everything --infer-dates`, `qsv frequency -l 20`,
`qsv sniff`, and `qsv validate` collapse that into a few hundred
bytes of structured summary the model can actually reason over:
column types, null counts, cardinality, top values, schema
violations, suspicious dates. Combined with `qsv slice -s 0 -e 5`
for a representative sample and `qsv search -s <col> <regex>` for
targeted spot-checks, an agent can characterise a 10 GB file in
one tool round-trip and propose a fix without ever ingesting the
raw bytes. The streaming and indexing model also means the agent
can iterate — re-running with a different filter — at constant
cost.

## Example invocations

```bash
# Profile every column: types, nulls, cardinality, quartiles, top values
qsv stats --everything --infer-dates orders.csv | qsv table

# Cheap row count and a 5-row sample (build an index first → O(1) slice)
qsv index orders.csv
qsv count orders.csv
qsv slice -s 0 -e 5 orders.csv | qsv table

# Frequency of values in a column, top 20
qsv frequency -s status -l 20 orders.csv

# Filter with a regex on one column, then select specific output columns
qsv search -s email '@example\.com$' users.csv \
  | qsv select id,email,signup_date \
  | qsv table

# Validate a CSV against a JSON Schema and write a row-level error report
qsv validate users.csv users.schema.json   # writes users.csv.invalid + .valid

# Join two CSVs (left join) using Polars for speed on big files
qsv joinp user_id users.csv user_id orders.csv --left > joined.csv

# Enrich a CSV by calling an HTTP API per row, with caching + 5 rps limit
qsv fetch --new-column geo --rate-limit 5 --cache-dir .qsv-cache \
  --url-template 'https://api.example.com/geo?ip={ip}' access.csv > geo.csv

# Run a Lua expression per row (no Python required)
qsv luau map total 'qty * price' line_items.csv > line_items_with_total.csv

# Convert to/from Excel without leaving the shell
qsv excel report.xlsx --sheet 'Q1' > q1.csv
qsv to xlsx report.xlsx q1.csv q2.csv q3.csv q4.csv
```
