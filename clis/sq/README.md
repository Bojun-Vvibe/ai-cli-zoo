# sq

> **`jq` for databases — a single Go binary that queries SQL DBs, CSV/TSV/JSON/Excel files, and pipes them in/out of each other with a unified jq-style language**
> — by Neil O'Toole. One CLI that connects to Postgres, MySQL,
> SQL Server, SQLite, plus flat files, exposes them as named
> *sources*, and lets you write `@sales | .user | where(.country
> == "DE") | .[0:10]` instead of context-switching between
> `psql`, `csvkit`, `jq`, and Excel. Pinned to **v0.50.0**
> ([LICENSE](https://github.com/neilotoole/sq/blob/master/LICENSE),
> MIT).

Source: <https://github.com/neilotoole/sq>

## TL;DR

`sq` is what you reach for when your "quick data question" spans
two systems: e.g. *"is the user in the Postgres `customers` table
also in the CSV the finance team just emailed me?"* The classic
answer is `psql -c "\copy ..." > tmp.csv && csvkit && jq ...`,
which means three tools, two intermediate files, and a schema
you have to re-discover each time. `sq` registers each system as
a *source* (`sq add postgres://prod/foo`, `sq add ./fin.csv`),
gives them short handles (`@prod`, `@fin`), and then runs a
single jq-flavoured query across them — including
cross-source `JOIN`s that it executes by streaming the smaller
side into a SQLite scratch DB and joining there.

The output side is symmetric: `sq` can emit JSON (compact,
array, lines), CSV, TSV, Markdown table, HTML, XML, XLSX, plus
a plain-text "table" mode that mirrors `psql`'s default. So a
realistic pipeline is `sq '@prod.users | where(.signup_at >
"2026-01-01")' --json | jq '.[] | .email' | sort -u` — but for
most queries you can stay inside `sq` end-to-end.

## Install

```bash
# Homebrew (macOS / Linux)
brew install neilotoole/sq/sq

# Go
go install github.com/neilotoole/sq@latest

# Release binary
curl -L https://github.com/neilotoole/sq/releases/download/v0.50.0/sq-0.50.0-linux-amd64.tar.gz \
  | tar -xz -C /usr/local/bin sq

# verify
sq version
```

`sq` writes its source registry to `~/.config/sq/sq.yml`.
Passwords are kept in the OS keychain by default
(`--password-from-keychain`); you can opt in to plaintext if
you really must (`sq config set source.password.cleartext true`).

## Usage

```bash
# 1) Register a Postgres DB and a local CSV, then preview both
sq add 'postgres://app:***@db.internal/prod' --handle @prod
sq add ./finance.csv --handle @fin
sq inspect @prod          # lists schemas, tables, row counts
sq inspect @fin           # detected column types from the CSV header

# 2) Cross-source JOIN: which paying users are missing from the CSV?
sq '@prod.users | join(@fin.customers, .email == .Email)
                | where(.plan == "pro" and .Email == null)
                | .[email, plan, signup_at]' --csv > missing.csv

# 3) One-shot ad-hoc query, output as a Markdown table to paste into a ticket
sq '@prod | .orders | where(.total > 1000) | .[customer_id, total]
          | sort_by(.total | desc) | .[0:20]' --markdown
```

## Niche & tradeoffs

`sq` sits in the gap between three established tools and
deliberately refuses to fully replace any of them. Compared to
[`usql`](../usql/) (a pure SQL REPL across many DBs), `sq` adds
flat files and a non-SQL query language but is a much weaker
*interactive* shell — `usql` still wins for ad-hoc DDL,
multi-statement transactions, or anything where you want
`\d table_name` muscle memory. Compared to [`csvkit`](../csvkit/)
or `xsv`, `sq` adds real DB sources and a richer query language
but is slower on pure CSV→CSV pipelines (`xsv` is hand-tuned for
that case and will out-throughput `sq` by an order of magnitude
on multi-GB files). Compared to [`duckdb`](../duckdb/), `sq` is
*much* less powerful at analytics — DuckDB's columnar engine,
Parquet support, and full SQL dialect dominate any
"give me a window function over a 50M-row file" workload — but
`sq` is far easier when the question is *operational* rather
than *analytical* and spans live DBs you don't want to
materialise into Parquet first.

The tradeoffs to pin into your runbook: (1) the jq-style
language is intentionally a *subset* of jq plus DB-aware verbs
(`join`, `groupby`, `where`); deeply nested JSON gymnastics are
not the goal, and you should pipe to real `jq` for those.
(2) Cross-source joins materialise the smaller side into SQLite
in `$TMPDIR` — fine for tens of thousands of rows, painful past
a few million; if both sides are large, do the join in the DB
that already has both. (3) The driver matrix is wider than it
is deep — Postgres, MySQL, SQL Server, SQLite, plus the file
formats are first class; Snowflake, BigQuery, ClickHouse are
not (yet). For those, stay with their native CLIs.

The right one-line frame: "**`sq` is the duct tape between your
production database and the spreadsheet someone just emailed
you.**" If your day involves more than one of {Postgres, MySQL,
SQLite, CSV, Excel, JSON} in the same hour, it earns its place
on `$PATH`. For pure-DB work prefer [`usql`](../usql/) or
[`harlequin`](../harlequin/); for pure-file analytics prefer
[`duckdb`](../duckdb/) or [`qsv`](../qsv/); for read-only
SQLite browsing the matching TUI is
[`harlequin`](../harlequin/) or [`litecli`](../litecli/).
