# sqlite-utils

> The Swiss-army CLI for SQLite: ingest CSV / JSON / NDJSON / Parquet
> into typed tables in one command, transform schemas in place, run
> queries with JSON output, and ship the resulting `.db` anywhere.
> Pinned to **v3.39** (commit `2026087a72c9af0412b7fa80fc2e03f81a373b79`,
> [LICENSE](https://github.com/simonw/sqlite-utils/blob/main/LICENSE), Apache-2.0).

Source: <https://github.com/simonw/sqlite-utils>

## TL;DR

`sqlite-utils` is the "make a SQLite database out of whatever I have
lying around" command. `sqlite-utils insert mydata.db events events.csv
--csv` creates the table, infers column types, and bulk-inserts in one
call. `sqlite-utils memory data.csv "select count(*) from stdin"` runs
SQL against a CSV without ever materializing a file. `sqlite-utils
extract` normalizes a denormalized column into a lookup table with a
foreign key. `sqlite-utils enable-fts events title body` adds an FTS5
index. Output of any query defaults to JSON (`--csv`, `--tsv`,
`--nl`, `--table` available), which is the point: it composes in shell
pipelines next to `jq`, `xsv`, and any LLM CLI in this catalog.

## Install

```bash
pip install sqlite-utils         # 3.39
# or
uv tool install sqlite-utils
brew install sqlite-utils
pipx install sqlite-utils

# verify
sqlite-utils --version
```

Python 3.8+. Pure-Python; bundles its own SQLite via `sqlite3` stdlib.

## One Concrete Example

```bash
# 1. ingest a CSV with header inference and chunked inserts
sqlite-utils insert logs.db requests access.csv --csv --detect-types

# 2. add a column derived from another (no ALTER TABLE typing)
sqlite-utils transform logs.db requests --add latency_ms integer

# 3. backfill it with SQL
sqlite-utils logs.db "update requests set latency_ms = cast(duration * 1000 as integer)"

# 4. full-text-search over the URL + UA
sqlite-utils enable-fts logs.db requests url user_agent --create-triggers

# 5. query, get JSON for the next pipe stage
sqlite-utils logs.db \
  "select url, count(*) c from requests where requests match 'mobile' group by url order by c desc limit 10"
# → [{"url": "...", "c": 1234}, ...]

# 6. one-shot SQL against a file with no DB at all
sqlite-utils memory access.csv "select status, count(*) from t group by 1"
```

The `memory` subcommand alone replaces a depressing amount of
ad-hoc Python — it loads each CSV / JSON arg as a table named `t`,
`t1`, `t2`, … and drops you straight into the query.

## Niche It Fills

**The shortest path from "I have a file" to "I have a queryable
database".** Everything else in the data-tooling lane in this catalog
either assumes you already have a DB (`harlequin`, `datasette`, `vanna`)
or is a heavyweight orchestration layer (`prefect`, `dagster`,
`metaflow`). `sqlite-utils` is the upstream loader and the schema
mutator — it's what produces the `.db` that the other tools open.
For LLM workflows, `sqlite-utils insert prompts.db runs out.json` is
the canonical way to land an LLM CLI's JSON output into a queryable
log without writing any Python.

## Vs Already Cataloged

- **Vs [`datasette`](../datasette/):** same author, same data model.
  `sqlite-utils` *writes* the database; `datasette` *serves* it.
  A common pipeline is `sqlite-utils insert ... && datasette serve
  ...`. They are designed to compose.
- **Vs [`llm`](../llm/):** `llm` (also Simon Willison) logs every
  prompt to a SQLite file at `~/.config/io.datasette.llm/logs.db`;
  `sqlite-utils` is exactly the tool to query and reshape that log
  (`sqlite-utils ~/.config/io.datasette.llm/logs.db "select model,
  count(*), sum(input_tokens) from responses group by 1"`).
- **Vs [`harlequin`](../harlequin/):** `harlequin` is interactive
  (TUI editor + results grid); `sqlite-utils` is non-interactive
  (one shell command per operation). Use `sqlite-utils` in scripts
  and CI, `harlequin` when you need to explore.

## Caveats

- Type inference on `--csv` is best-effort and will pick `TEXT` when in
  doubt. For mixed columns (`"42"` and `"forty-two"` in the same
  column), pass `--detect-types` and verify with `sqlite-utils schema`.
- `transform` rebuilds the table under the hood (SQLite has no full
  `ALTER`), so on a multi-GB table it's slow and needs free disk
  equal to the table size. Run it during a maintenance window.
- `enable-fts` with `--create-triggers` keeps the FTS table in sync on
  insert/update/delete — but writes are now ~2× slower. For
  write-heavy workloads, rebuild FTS periodically with `populate-fts`
  instead.
- The CLI talks to SQLite only; for Postgres / MySQL ingestion the
  sister project is `db-to-sqlite`. `sqlite-utils` will not connect to
  a server-based database.
