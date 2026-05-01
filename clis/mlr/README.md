# mlr (Miller)

- **Repo:** https://github.com/johnkerl/miller
- **Version:** v6.18.1 (tagged 2026-04-19; the v6.x line is the current stable series)
- **License:** BSD-2-Clause "Views" — see [`LICENSE.txt`](https://github.com/johnkerl/miller/blob/main/LICENSE.txt)
- **Language:** Go (since v6; the legacy v5.x line was C)
- **Install:** `brew install miller` · `apt install miller` · `pacman -S miller` · `dnf install miller` · binary name is `mlr` · prebuilt binaries on the GitHub release page for Linux/macOS/Windows/BSD

## Overview

`mlr` is a single-binary stream processor for **name-indexed
tabular data** — CSV, TSV, JSON / JSON-Lines, Parquet,
Pretty-Print, key-value-pair (DKVP), positionally-indexed,
and fixed-width formats — with the same verb-and-pipeline
ergonomics as `awk`, `sed`, `cut`, `join`, and `sort` combined.
The defining property is that records are dictionaries keyed
by header name, not positional fields, so a pipeline like
`mlr --c2j cat input.csv | mlr --j2c put '$total=$qty*$price' then stats1 -a sum -f total -g region` reads
CSV in, emits JSON, mutates a record with a typed expression,
and aggregates by group — all in one process, streaming row by
row with constant memory. v6 added a real expression language
(`mlr put`/`mlr filter` with maps, arrays, regex, datetime,
HOF, user-defined functions) and gained the DKVPX binary
format in v6.18 for fast intermediate spooling. Format
conversion is first-class: `--c2j`, `--j2c`, `--t2p`, `--icsv
--ojsonl`, etc. are short flags rather than separate tools.

## Niche

**Schema-aware tabular wrangling at the shell**. It sits
between "spreadsheet" and "load it into pandas / DuckDB":
operates on streams (no `LOAD DATA`), handles ragged headers
and heterogeneous record shapes, and is one binary with no
runtime dependency. Reach for it when the data has named
columns and you want to express the transform as a Unix
pipeline of named verbs (`cat`, `cut`, `head`, `tail`, `sort`,
`uniq`, `join`, `group-by`, `stats1`, `stats2`, `top`,
`reshape`, `nest`, `unsparsify`, `tee`).

## When to use

- Converting between CSV ↔ TSV ↔ JSON-Lines ↔ Parquet ↔
  fixed-width ↔ pretty-print without a script.
- Joining two CSVs on a key column from the shell
  (`mlr --csv join -j id -f left.csv right.csv`).
- Group-by aggregations on multi-GB files that don't fit in
  memory: `mlr --csv stats1 -a sum,mean,p99 -f latency -g region huge.csv`.
- Mid-pipeline schema rewrites: rename fields with regex,
  fill in missing columns (`unsparsify`), explode nested JSON
  (`nest --explode`), or pivot wide↔long (`reshape`).
- Fast typed scripting where `awk` runs out of language —
  Miller's DSL has typed numerics, datetime arithmetic,
  regex, maps, arrays, and `tee` to multiple sinks.

## When NOT to use

- The data is genuinely relational and you need indexes,
  joins across many tables, or window functions over big
  data — load it into [`duckdb`](../duckdb/) (already in the
  zoo) or a real DBMS.
- Pure column projection / filtering of TSV at maximum
  throughput on a single file → [`xsv`](../xsv/) (Rust, faster
  on TSV/CSV-only workloads where you don't need format
  conversion).
- One-shot "find lines matching X" jobs — `rg` / `grep` is
  the right tool, not a record-aware processor.
- You want an interactive viewer with sort/filter/freeze-pane
  in a TUI → `visidata` (not in the zoo as of this entry,
  different niche). `mlr` is batch / pipeline-first.

## Comparison vs alternatives in zoo

- [`jq`](../jq/) / [`gojq`](../gojq/) / [`yq`](../yq/) —
  JSON / YAML structural transforms; reach for them when the
  input is a single JSON document or YAML tree. `mlr` shines
  when records are *streamed* and tabular conversions are
  involved.
- [`xsv`](../xsv/) — Rust, CSV-only, very fast for slicing,
  indexing, and joining. Pick it when the workload is
  read-only CSV and throughput dominates; pick `mlr` when the
  workload mixes formats or needs a real expression language.
- [`dasel`](../dasel/) — selector-language for JSON / YAML /
  TOML / XML; complementary, not overlapping — `dasel` does
  one document, `mlr` does streams of records.
- [`gron`](../gron/) — flattens JSON to greppable lines for
  ad-hoc inspection; complementary at the discovery stage,
  `mlr` for the actual transform.
- [`htmlq`](../htmlq/) — selector for HTML; out of scope here
  but listed because it's the structural-extractor cousin.

## Why it earns a slot in an AI-native workflow

LLM agents that touch real-world datasets spend most of
their tool budget on the same boring stuff: convert this
CSV to JSON-Lines so the model can read it, group by
`session_id` and emit one row per session, drop the PII
columns before sending to the API, fan a Parquet file out
to one JSON-Lines shard per partition. `mlr` collapses each
of those into one verb-pipeline, which is much cheaper for
an agent to compose and verify than a Python snippet (no
venv, no pandas import, no schema-inference surprises). The
verb names are stable and well-documented, so a model can
emit a correct pipeline zero-shot most of the time.

## Example invocations

```bash
# CSV → pretty-printed for human inspection
mlr --c2p cat data.csv | head

# CSV → JSON-Lines, dropping a PII column on the way out
mlr --c2j put 'unset $email,$phone' then cat data.csv > data.jsonl

# Group-by aggregate with multiple stats
mlr --csv stats1 -a sum,mean,p50,p99 -f latency_ms -g region requests.csv

# Streaming join of two CSVs on a key column
mlr --csv join -j user_id -f users.csv -- then cat events.csv

# Pivot long → wide
mlr --csv reshape long-to-wide -k metric -v value metrics.csv

# Typed expression: derive a column, filter, then take top-10
mlr --csv put '$margin = ($price - $cost) / $price' \
    then filter '$margin > 0.2' \
    then top -n 10 -f margin sales.csv

# Convert Parquet → JSON-Lines
mlr --iparquet --ojsonl cat snapshot.parquet
```
