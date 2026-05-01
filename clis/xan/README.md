# xan

> **A Rust-native CSV toolkit that streams transformations,
> aggregations, joins, plots, and stats over CSV / TSV files
> too big to open in a spreadsheet** — `xan` is a friendly
> rewrite of `xsv` (which has been unmaintained since 2018) by
> the SciencesPo médialab, with added `select` / `map` / `agg`
> mini-language, terminal plotting, parallel processing, and
> full streaming so a 30 GB CSV processes in constant memory.
> Pinned to **v0.57.1**
> ([LICENSE-MIT](https://github.com/medialab/xan/blob/master/LICENSE-MIT)
> and [UNLICENSE](https://github.com/medialab/xan/blob/master/UNLICENSE),
> dual MIT / Unlicense).

Source: <https://github.com/medialab/xan>

## TL;DR

`xan` is what you reach for when a CSV is too large for Excel
or pandas-on-laptop, but you do not want to spin up DuckDB and
write SQL just to "show me the top 10 user_ids by request count
from this 12 GB access log". It reads CSV/TSV/whatever-delimiter
in a streaming fashion, every subcommand emits CSV on stdout, so
pipelines compose: `xan select user_id,bytes data.csv | xan
filter 'bytes > 1000000' | xan agg 'sum(bytes) as total,
count() as hits' -g user_id | xan sort -R total | xan view -l 10`.
The mini-language across `select` / `filter` / `map` / `agg` is
small (numeric ops, string ops, `case`, `if`, basic aggregates)
and consistent — once you learn `filter '<expr>'` you also know
`map 'expr as colname'` and `agg 'expr as colname' -g groupcol`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install xan

# Cargo (any platform with Rust 1.74+)
cargo install xan

# pre-built binary release (Linux x86_64 / aarch64, macOS arm64,
# Windows x86_64)
curl -LO "https://github.com/medialab/xan/releases/download/0.57.1/xan-x86_64-unknown-linux-musl.tar.gz"
tar xzf xan-*.tar.gz
sudo mv xan /usr/local/bin/

# verify
xan --version    # xan 0.57.1
```

No daemon. No config file. One binary, ~12 MB, links statically
against musl on Linux.

## License

Dual MIT / Unlicense — see
[LICENSE-MIT](https://github.com/medialab/xan/blob/master/LICENSE-MIT)
and
[UNLICENSE](https://github.com/medialab/xan/blob/master/UNLICENSE).
The `COPYING` file documents the dual-licensing intent. Pick
whichever is convenient for your downstream — both are
maximally permissive.

## One Concrete Example

```bash
# 1. peek at a large CSV (constant-memory; reads only the head)
xan view -l 10 access.csv
xan headers access.csv               # just the column names + types
xan count access.csv                  # exact row count, streaming

# 2. project columns + filter rows + sort, all streaming
xan select user_id,path,bytes,status access.csv \
  | xan filter 'status >= 400 and bytes > 0' \
  | xan sort -s bytes -R \
  | xan view -l 20

# 3. group-by aggregation: top URL paths by total bytes
xan agg 'count() as hits, sum(bytes) as total_bytes' \
        -g path access.csv \
  | xan sort -s total_bytes -R \
  | xan view -l 25

# 4. add a derived column with `map` (mini-language)
xan map 'bytes / 1024.0 as kib, upper(method) as method_u' access.csv

# 5. join on key (streaming-hash join)
xan join user_id access.csv user_id users.csv \
  | xan select user_id,country,path,bytes \
  | xan agg 'sum(bytes) as bytes' -g country \
  | xan sort -s bytes -R | xan view

# 6. terminal plot (histogram, scatter, bar, line) without ever
#    leaving the shell
xan agg 'count() as hits' -g status access.csv \
  | xan plot bar status hits

xan plot hist bytes access.csv --bins 50

# 7. stats: cardinality, nulls, type, mean / median / quantiles
xan stats access.csv
xan stats -s bytes,duration -A access.csv   # all stats per column

# 8. parallel processing: --parallel splits work over CPU cores
xan filter --parallel 'status >= 500' huge.csv > errors.csv

# 9. progress while crunching (xan reads stdin if no file)
zcat dump.csv.gz | pv -l | xan filter 'country == "FR"' > fr.csv

# 10. write to spreadsheet-friendly tab format for the analyst
#     down the hall
xan tsv access.csv > access.tsv
```

## Niche It Fills

**A `xsv` successor with a real expression language and
streaming joins/aggregates, that lives in the shell instead of
in a Python interpreter.** The catalog already has [`miller`](
../miller/) (mlr) and [`csvkit`](../csvkit/) and [`qsv`](../qsv/)
and [`duckdb`](../duckdb/) — they overlap, but `xan`'s niche is
the specific spot of "I want one tiny binary, one consistent
mini-language across select/filter/map/agg/join, and built-in
terminal plotting", at >100 MB/s on a laptop and constant memory
on multi-GB inputs. It is opinionated about CSV (not arbitrary
columnar), single-machine (no Spark/Dask), and shell-pipeline
native (every command reads CSV on stdin and writes CSV on
stdout).

## Why use it

Three things `xan` does that picking one of the existing CSV
tools does not, that explain why it earned a separate entry:

1. **One mini-language across every verb.** `xan filter '<expr>'`
   and `xan map 'expr as col'` and `xan agg 'expr as col' -g k`
   all parse the same expression grammar (numeric, string, regex,
   `case`/`if`, aggregates `sum`/`count`/`avg`/`min`/`max`/
   `quantile`/`first`/`last`). You don't context-switch between
   `awk` syntax for filters, `sed` syntax for substitutions, and
   SQL for aggregations. Once you've written a filter, you can
   reuse half the expression as a `map`.
2. **Streaming joins + aggregations + sort.** `xan join`,
   `xan agg`, and `xan sort` all stream and spill to disk when
   needed; you don't have to load either side into RAM. That
   makes "join 12 GB of access logs against 200 MB of users.csv
   and aggregate" a one-liner that finishes on a 16 GB laptop,
   without DuckDB, without writing SQL, and without leaving the
   shell pipeline.
3. **Terminal plots as part of the pipeline.** `xan plot bar`,
   `xan plot hist`, `xan plot scatter`, `xan plot line` render
   directly in the terminal (Unicode block characters) on the
   output of any preceding `xan` command. The "see the
   distribution before deciding the next filter" loop happens
   in the shell, in seconds, without spawning a Jupyter kernel.

For an LLM-CLI workflow, `xan` is the substrate that turns a
`requests`-fetched CSV into machine-checkable summary numbers
(`xan agg 'count() as n, sum(amount) as total' -g country |
xan to json`) before passing them as context to the next prompt
— much cheaper than streaming the raw CSV through the model.

## Vs Already Cataloged

- **Vs [`miller`](../miller/) (mlr):** Miller is the dean of
  shell-native record processors and handles many more formats
  (CSV, TSV, JSON, JSON-lines, DKVP, NIDX, pprint). Its DSL
  (`mlr put`, `mlr filter`) is more powerful (real control flow,
  user-defined functions, complex regex). Tradeoffs: miller is
  C and has decades of sharp edges; xan is Rust, narrower scope
  (CSV-shaped only), and more uniform across verbs. Pick miller
  for multi-format pipelines and complex per-record logic; pick
  xan for "CSV in / CSV out" with stats and plots, and for the
  single-binary install on a fresh machine.
- **Vs [`qsv`](../qsv/):** qsv is the most feature-complete xsv
  fork (geocoding, sniffing, fuzzy join, polars backend for
  some commands, Python embedded for `apply py`). Tradeoffs: qsv
  is bigger (~30 MB binary, more dependencies), more invasive
  features; xan is smaller and stays inside the "Unix tool"
  envelope. Pick qsv for the long tail of obscure CSV chores
  (timezone-aware date parsing, `geocode reverse`, `fuzzy join`);
  pick xan for the everyday "filter, group, aggregate, sort,
  plot" loop.
- **Vs [`csvkit`](../csvkit/):** csvkit is Python and uses
  in-memory pandas-shaped processing for most commands; great
  ergonomics (`csvsql`, `csvlook`), poor performance on multi-GB
  inputs because it loads the file. Pick csvkit for small files
  + ad-hoc SQL queries (`csvsql --query …`); pick xan when the
  file does not fit in RAM.
- **Vs [`duckdb`](../duckdb/):** DuckDB is the heavy-hitter for
  analytical CSV/Parquet — `SELECT … FROM 'file.csv'` queries a
  10 GB CSV at memory-mapped speed and gives you full SQL with
  joins, window functions, and CTEs. Tradeoffs: you write SQL,
  not shell expressions; the binary is bigger; output is a
  table renderer, not a CSV pipeline element by default. Pick
  DuckDB when you want SQL ergonomics or window functions or
  Parquet support; pick xan when you want a streaming Unix
  pipeline that you can `| grep` / `| head` between stages.
- **Vs `xsv` (the original, not cataloged because abandoned):**
  Same scaffolding, but xsv has been unmaintained since 2018
  and lacks `map` / `agg` / `plot` / `parallel`. xan is the
  intended successor.

## Caveats

- **CSV-shaped only.** No native JSON / Parquet / Arrow input.
  `xan from json` exists for line-delimited JSON, but for
  Parquet you should pipe through DuckDB first or use qsv.
- **Schema is inferred, not declared.** Numeric columns are
  detected by sampling; mixed-type columns become strings and
  arithmetic on them fails at runtime. Pre-`xan select` your
  numeric columns with confidence in their type, or use
  `xan map 'parseint(col) as col'` to coerce.
- **The mini-language is not Turing-complete.** No loops, no
  user-defined functions. Multi-step transformations live as
  multi-stage pipelines (`xan map ... | xan filter ...`). For
  anything more elaborate, `xan map` can shell out via
  `xan map -t` (run a template per row), but at that point
  miller or duckdb is usually the right tool.
- **Sort spills to disk under `$TMPDIR`.** A 30 GB CSV sorted
  in one pass writes ~30 GB of temp files. Make sure `$TMPDIR`
  has space (`xan sort -T /var/tmp` to relocate).
- **`xan plot` is approximate.** Unicode block-character plots
  are great for "what's the shape" but you should not embed them
  in a report. For publication graphics, pipe to a real plotter
  (`xan to json | python plot.py`).
- **API still evolving (0.x).** The maintainers explicitly call
  out that flag names and expression syntax may shift before
  1.0; pin a version in CI rather than relying on `latest`.
