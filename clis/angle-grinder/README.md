# angle-grinder

> **A streaming SQL-ish query language for log files** — pipe raw
> log lines (or JSON, or k=v) into `agrind` and apply a chain of
> `parse`, `where`, `fields`, `json`, `sum`, `count`, `pct`, `sort`
> operators that look like a flat SQL pipeline, getting an
> aggregated table out the other end. The point: you do not need to
> ship logs to Splunk or Elastic to ask "what's the p99 latency by
> route over the last 1M requests in this gzip file". Pinned to
> **v0.19.6** (commit `e98ad5c286f2415ef933acc4563649bd26cea684`,
> [LICENSE](https://github.com/rcoh/angle-grinder/blob/master/LICENSE),
> MIT).

Source: <https://github.com/rcoh/angle-grinder>

## TL;DR

The classic UNIX way to ask "top 10 slow endpoints" of an access
log is a 3-line `awk | sort | uniq -c | sort -rn | head` chain
that you re-derive every time. `angle-grinder` (`agrind` on the
CLI) replaces that with a single, composable query that reads
left-to-right like a stream pipeline:

```text
* | json | where status >= 500 | count by route | sort by _count desc | limit 10
```

Each `|` stage is a typed operator over the row stream. `parse`
extracts named fields with a glob-like syntax (`parse "* * *" as
ip, ts, rest`), `json` walks JSON, `logfmt` walks `k=v`,
`fields` projects, `where` filters, `sum`/`count`/`pct`/`p50`
aggregate, `sort`/`limit` finalize. The output is a pretty
table by default, or `--output-format=json` for piping onward.
There is no index, no daemon, no schema — `agrind` is a single
~10 MB Rust binary that streams stdin and forgets the row when
it is done with it.

## Install

```bash
# Homebrew
brew install angle-grinder

# Cargo
cargo install ag

# From source
git clone https://github.com/rcoh/angle-grinder
cd angle-grinder
cargo build --release
./target/release/agrind --version    # agrind 0.19.6

# verify
agrind --version
```

A first query against an nginx access log:

```bash
zcat access.log.gz | agrind '
  * | parse "* * * [*] \"* * *\" * *" as ip, _, _, ts, method, path, _, status, bytes
    | where status >= "500"
    | count by path
    | sort by _count desc
    | limit 10
'
```

Latency percentiles from JSON logs:

```bash
agrind '* | json | p50(latency_ms), p99(latency_ms) by route' < requests.jsonl
```

## Why it's worth a slot in the zoo

There is a real ladder of "ask questions of log files" tools:
`grep` → `awk` → `jq` → spinning up Loki / Elastic /
ClickHouse. `agrind` lives precisely between `jq` and "build
real infrastructure": it is grep-fast on a single file, but
gives you the SQL-ish vocabulary (`count by`, `pct`, `sort`)
that you would otherwise have to leave the terminal for. It is
particularly good for incident response — you have a 4 GB
`gunzip`-able log on a single host, you want the answer in 30
seconds, and you do not want to ingest anything anywhere.

## Where it sits

- vs `awk`: same niche (one-shot stream queries), but `agrind`
  has a typed row model and built-in aggregation operators, so
  you do not write your own `arr[$5]++; END { for (k in arr) ... }`.
- vs `jq`: `jq` is a JSON transformer; `agrind` is a query
  engine that *includes* JSON parsing. If your input is uniform
  JSON and you want to *reshape* it, use `jq`. If you want to
  *aggregate* across many lines, use `agrind`.
- vs `lnav`: `lnav` is an interactive log viewer with SQLite as
  the query backend; great for exploration. `agrind` is for
  scripted, one-shot pipelines.
- vs `vector` / `fluent-bit`: those are *log shippers* with a
  query DSL bolted on; they assume you are running a daemon.
  `agrind` is a single CLI invocation against a file.
- vs ClickHouse / DuckDB on logs: those scale far higher and do
  joins, but you have to load data first. `agrind` does not.

## Footguns

- **The query language is not SQL.** It looks SQL-ish but is its
  own thing — `where` uses `==` for equality, string literals
  are double-quoted, `count` is `count` not `count(*)`, and
  there are no joins. Read the
  [operators reference](https://github.com/rcoh/angle-grinder#available-operators).
- **`parse` is glob-style, not regex.** `parse "* * *" as a, b,
  c` greedy-matches by spaces. For real regex extraction use
  `parse regex "..."`. Mixing the two confuses a lot of
  newcomers.
- **No index, no resume.** It re-reads the whole input every
  time. On a 50 GB rotated log directory this is fine for one
  query but painful for ten — pre-filter with `rg` first.
- **Aggregations are streaming but bounded.** `count by user_id`
  on a high-cardinality field can blow memory; the docs note
  that aggregation state is held in a HashMap. Pre-filter or
  bucket coarsely.
- **Type inference is light.** Numeric comparisons sometimes
  need an explicit cast (`status as int >= 500`) — otherwise
  string comparison can produce surprising results
  (`"500" < "9"`).
- **No structured output schema.** `--output-format=json` emits
  a row stream, but field names depend on your query. Downstream
  tools should expect `{}` records keyed by your `as` names and
  the aggregator names (`_count`, `p99`, …).
