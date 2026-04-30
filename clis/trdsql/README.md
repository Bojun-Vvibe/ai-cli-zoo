# trdsql

- **Repo:** https://github.com/noborus/trdsql
- **Latest version:** v1.1.0 (2025)
- **HEAD on `master`:** `27e8c23`
- **License:** MIT — [LICENSE](https://github.com/noborus/trdsql/blob/master/LICENSE)
- **Category:** data-format conversion / SQL on files

A **single Go binary that runs SQL on plain files** — CSV, TSV,
LTSV, JSON, JSON-Lines, YAML, and TBLN — and emits the result in any
of the same formats. Internally each input is auto-detected (or
forced via `-i<format>`), loaded into an in-memory **SQLite** or
**MySQL** / **PostgreSQL** backing store as a temporary table named
after the file, and your `SELECT … FROM <file>` query runs against
that engine. Because the engine is real SQL, you get joins,
aggregates, window functions, sub-queries, `CASE`, `WITH`, and
`ORDER BY` for free, on data that lives only as files on disk.
Output formatting is parallel: `-o<format>` covers CSV / TSV / LTSV /
JSON / JSON-Lines / YAML / TBLN / Markdown / ASCII-table / vertical,
so the same query can drive a pipeline stage *or* a Markdown report.

## Install

```bash
# macOS
brew install noborus/tap/trdsql

# Linux / macOS / Windows — Go install
go install github.com/noborus/trdsql/cmd/trdsql@latest

# Linux — prebuilt binary
curl -L -o trdsql.tar.gz https://github.com/noborus/trdsql/releases/download/v1.1.0/trdsql_v1.1.0_linux_amd64.tar.gz
tar xzf trdsql.tar.gz && sudo mv trdsql_v1.1.0_linux_amd64/trdsql /usr/local/bin/

# verify
trdsql -version
```

## One usage example

```bash
# Join a CSV of users with a JSON-Lines events file, group, render Markdown:
trdsql -ih -omd "
  SELECT u.name, COUNT(*) AS events, MAX(e.ts) AS last_seen
  FROM   users.csv u
  JOIN   events.jsonl e ON e.user_id = u.id
  GROUP  BY u.name
  ORDER  BY events DESC
  LIMIT  10
"
```

The `-ih` flag treats the first CSV row as a header (so the columns
become `u.name`, `u.id`, …); `-omd` renders the result as a GitHub
Markdown table you can paste straight into a PR description.

## When to pick `trdsql` (vs alternatives)

- Pick **`trdsql`** when the task is *literally* "I want to write a
  multi-file SQL query against CSV / JSON / YAML on disk and get a
  CSV / JSON / Markdown table back". One binary, no schema setup, no
  database to start, joins across heterogenous formats in one query
  (CSV ⨝ JSON ⨝ YAML), and the SQLite / MySQL / PostgreSQL backend
  is selectable per-invocation when you need engine-specific dialect.
- Pick [`csvkit`](../csvkit/) (specifically `csvsql`) when you are
  already on a Python stack and want the broader `csvcut` /
  `csvgrep` / `csvstat` family alongside the SQL runner; `csvkit`
  is CSV-first and slower on JSON / YAML.
- Pick [`miller`](../miller/) (`mlr`) when the workload is
  per-record streaming transforms (`mlr put`, `mlr filter`,
  `mlr stats1`) on huge files where an in-memory SQL engine would
  blow the heap; miller is line-by-line, trdsql is "load then
  query".
- Pick [`jq`](../jq/) / [`jaq`](../jaq/) when the input is *only*
  JSON and the output is *only* JSON — they win on raw JSON
  ergonomics and have no SQL learning curve.
- Pick [`yq`](../yq/) when the workload is YAML mutation (in-place
  edits, comment preservation), not querying.
- Pick [`datasette`](../datasette/) when you want to publish the
  same dataset as a *browseable web UI* with a JSON API; `trdsql`
  is for one-shot terminal queries, `datasette` is for "share this
  CSV with the team".
- Pick [`visidata`](../visidata/) when you want to *interactively*
  explore an unknown CSV / JSON file before deciding what to query;
  `trdsql` assumes you already know the SQL you want to run.
