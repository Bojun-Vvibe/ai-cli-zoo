# q-text-as-data

> **Run SQL directly against CSV / TSV / delimited files** — and
> against multi-file SQLite DBs — from the shell. No load step,
> no schema declaration, no database server: `q "SELECT c1,
> COUNT(*) FROM ./access.tsv GROUP BY c1"` and you get a result.
> Pinned to **v3.1.6**
> ([LICENSE](https://github.com/harelba/q/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/harelba/q>

## TL;DR

`q` (the binary is `q`, the project is "q - Text as Data") is a
Python program that exposes any delimited text file as a SQLite
table and then runs your SQL query against it. It auto-detects
delimiter and types, supports `JOIN` across files, has caching
for re-querying the same large file, and emits results to stdout
in the format you ask for (TSV, CSV, formatted columns). It also
queries `.sqlite` files directly so you can mix CSVs and SQLite
tables in the same query. Single-binary install, no daemon.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install q

# Debian / Ubuntu (.deb from releases)
curl -L -o q.deb \
    https://github.com/harelba/q/releases/download/v3.1.6/q-text-as-data-3.1.6-1.x86_64.deb
sudo dpkg -i q.deb

# RPM (Fedora / RHEL)
sudo rpm -i \
    https://github.com/harelba/q/releases/download/v3.1.6/q-text-as-data-3.1.6.x86_64.rpm

# from source
git clone https://github.com/harelba/q && cd q
pip install -e .

# verify
q --version       # 3.1.6
```

## License

GPL-3.0 — see
[LICENSE](https://github.com/harelba/q/blob/master/LICENSE).
Strong copyleft: redistributing modified `q` requires sharing
source under the same license. Using `q` as an end-user CLI in
your shell scripts / pipelines does *not* infect your scripts
(they merely *call* `q`, they don't link against it). Embedding
`q`'s source into your own product is the case to avoid; for
that, prefer one of the MIT-licensed alternatives below.

## One Concrete Example

```bash
# 1. headerless CSV: count rows per status code in a web log
q -d ' ' -t \
    "SELECT c9, COUNT(*) FROM ./access.log GROUP BY c9 ORDER BY 2 DESC"
# c9 = ninth space-separated column = HTTP status in a default
# combined log; -t = output as TSV.

# 2. CSV with header — name your columns
q -H -d ',' \
    "SELECT category, SUM(amount) AS total
     FROM ./expenses.csv
     WHERE date >= '2026-01-01'
     GROUP BY category
     ORDER BY total DESC"

# 3. JOIN two files (orders + customers) on a shared key
q -H -d ',' \
    "SELECT c.country, SUM(o.total) AS revenue
     FROM ./orders.csv o
     JOIN ./customers.csv c ON o.customer_id = c.id
     GROUP BY c.country
     ORDER BY revenue DESC
     LIMIT 10"

# 4. mix a CSV with a real SQLite DB
q -H -d ',' \
    "SELECT u.email, COUNT(e.id) AS events
     FROM ./users.csv u
     JOIN ./events.sqlite:::events e ON e.user_id = u.id
     GROUP BY u.email"

# 5. cache the parsed table so repeated queries are instant
q -H -d ',' -C readwrite \
    "SELECT COUNT(*) FROM ./huge-10gb-export.csv"
# first run parses + writes ./huge-10gb-export.csv.qsql;
# subsequent queries use the .qsql cache directly.
```

## Niche It Fills

**The "I have a CSV and one SQL question, not a project" gap.**
The two existing options are both heavy: spin up SQLite (`.mode
csv`, `.import`, then query, then exit), or load the file into a
DataFrame (`pandas.read_csv` + `df.query` or `df.groupby`). Both
require a few minutes of setup per ad-hoc question. `q` lets you
write the SQL directly against the file path on the command line
and get an answer in one shot — the same place you already use
`awk` / `cut` / `sort | uniq -c`, but with `JOIN` and `GROUP BY`
that those tools don't have.

## Why use it

Three concrete things `q` makes pleasant:

1. **Real SQL engine, not toy.** The backend is SQLite, so window
   functions (`ROW_NUMBER() OVER (PARTITION BY …)`), CTEs (`WITH
   recent AS (…)`), `GROUP BY ROLLUP`-style subqueries, and `LIKE`
   / `REGEXP` all work. You're not stuck with awk's "associative
   array of counters" pattern.
2. **Multi-file `JOIN` without a schema.** Two CSVs, a TSV, and a
   `.sqlite` file in the same query. The "schema" is the file
   path. There is no DDL, no `CREATE TABLE`, no migration. For
   one-off analysis across exports from three different systems
   that's a real win.
3. **Cache file is a portable artifact.** `-C readwrite` writes
   `<file>.qsql` next to the input; that file is a regular
   SQLite database you can also open with `sqlite3` directly, so
   the same artifact serves both `q` queries and downstream
   tools. Lets you turn an ad-hoc CSV into a real DB without a
   separate "ingest" step.

For an LLM-CLI workflow, `q` is the **structured-question step**
on top of unstructured CSV exports: an agent generates a SQL
query against `./report.csv`, you run it via `q`, the result is
small and quotable for the next prompt. The agent never has to
load the file into memory or write a custom parser.

## Vs Already Cataloged

- **Vs [`xsv`](../xsv/):** `xsv` is the fast Rust toolkit for
  *row-and-column transformations* on CSV (`xsv select`,
  `xsv join`, `xsv frequency`). Use `xsv` for "give me columns
  3, 5, 7 sorted by 3 deduplicated"; use `q` when you want
  expression-level SQL (`CASE WHEN`, window functions, sub-
  queries) or when you need to join across more than two files
  fluently.
- **Vs [`miller`](../miller/) / `mlr`:** Miller is a streaming
  CSV/TSV/JSON record processor with its own DSL (`mlr put`,
  `mlr stats1`). Better than `q` for *streaming* (no load,
  constant memory) and for nested JSON. Worse than `q` for
  arbitrary SQL — Miller's DSL doesn't have `JOIN ... ON`
  across many files in one expression.
- **Vs [`dasel`](../dasel/) / [`jq`](../jq/):** Those are
  selectors / transformers for tree-structured data (JSON /
  YAML / TOML). `q` is for tabular data. Different problem
  shape; choose by input format.
- **Vs `sqlite3 -csv`:** The plain `sqlite3` shell can `.import`
  a CSV and then run any SQL. It works, but is interactive-
  modal: you're inside the SQLite REPL, you have to re-`.import`
  per session, and you can't trivially pipe the result back
  out. `q` collapses that into one CLI invocation.
- **Vs `duckdb`:** DuckDB has been eating this niche from the
  performance angle (columnar, vectorized, reads CSV/Parquet
  natively). For >1 GB files DuckDB will be much faster. Pick
  DuckDB when you care about throughput on large files, pick
  `q` when you want one tiny pip-installable binary with no
  C++ dependency and your files are <100 MB.

## Caveats

- **GPL-3.0.** End-user CLI use is fine; embedding `q`'s source
  into a redistributed product is the case the GPL targets.
  For embedding, use `xsv` (Unlicense / MIT) or DuckDB (MIT).
- **Python startup cost.** Every invocation pays Python
  interpreter startup (~150 ms). For tight inner loops calling
  `q` thousands of times, batch your queries instead.
- **Type detection is heuristic.** Numbers-that-look-like-numbers
  become numbers; if column 5 has `007` it becomes `7`. Use
  `--as-text` or quote / cast in SQL when leading zeros, phone
  numbers, or postal codes matter.
- **Memory model is "load into SQLite then query".** A 10 GB
  CSV will produce a >10 GB SQLite database during the load.
  Use `-C readwrite` so you only pay that once, or move to
  DuckDB for true out-of-core analytics.
- **Latest tag is 2021.** v3.1.6 is stable and widely packaged,
  but new features land slowly. For active development on the
  same niche, watch DuckDB's CSV reader.
