# miller

- **Upstream:** https://github.com/johnkerl/miller
- **Version:** v6.18.1 (2026-04-19)
- **License:** BSD-2-Clause — https://github.com/johnkerl/miller/blob/main/LICENSE.txt (SPDX: `BSD-2-Clause`)

## What it does

Miller (`mlr`) is like `awk`, `sed`, `cut`, `join`, and `sort` for
name-indexed data — CSV, TSV, JSON, JSON Lines, Parquet, and a handful of
other tabular formats. Records are addressed by field name rather than
column number, so pipelines stay readable when columns are reordered or
renamed. It can stream gigabytes without loading the whole file and exposes
a small DSL for filters, computed fields, group-by aggregations, and joins.

## Example

```sh
# Pretty-print a CSV
mlr --c2p cat data.csv

# Filter and project columns, then sort
mlr --csv filter '$status == "ok"' then cut -f id,latency_ms then sort -nr latency_ms data.csv

# Group-by aggregation across JSON Lines
mlr --ijsonl --opprint stats1 -a mean,p95 -f latency_ms -g region events.jsonl
```
