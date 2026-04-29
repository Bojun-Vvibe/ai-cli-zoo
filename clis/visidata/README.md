# visidata

- **Repo:** https://github.com/saulpw/visidata
- **Version:** v3.3 (commit `d030fded3ff2826cea7e7d1f0278253898e379fc`,
  released 2025-09-08)
- **License:** GPL-3.0 ([LICENSE.gpl3](https://github.com/saulpw/visidata/blob/develop/LICENSE.gpl3))
- **Language:** Python 3
- **Install:** `pipx install visidata` · `brew install visidata` ·
  `pip install visidata` · `apt install visidata` (Debian / Ubuntu)

## What it does

`vd` (VisiData) is a terminal spreadsheet that opens **anything that
looks like tabular data** — CSV, TSV, JSON, JSON-lines, Parquet, Arrow,
XLSX / XLS / ODS, SQLite / PostgreSQL / MySQL connections, HDF5, Pandas
pickles, HTML tables, Markdown tables, fixed-width text, log files
parsed via regex, even *directories* (each file becomes a row) and
*shell processes* (`ps` / `lsof` output). Once a sheet is open you get
keystroke-driven exploration that maps directly onto data-engineering
verbs: `[` / `]` sort by the current column ascending / descending,
`|` and `\` filter rows by regex (keep / drop), `+` aggregate the
current column with a chosen function (`sum` / `mean` / `min` / `max`
/ `distinct` / `count`), `F` build a frequency table of the current
column (one keystroke = `SELECT col, COUNT(*) GROUP BY col ORDER BY 2
DESC`), `gw` join two sheets on key columns, `=` add a derived column
from a Python expression that sees row attributes (`row.price *
row.qty`), `Ctrl+S` save the current sheet to *any* of the supported
formats — so VisiData doubles as a format converter (`vd
data.json -b -o data.csv` is non-interactive `json → csv`). Every
keystroke is logged into a *command log* (`Ctrl+D` to view, `Ctrl+Y`
to save as `.vd` / `.vdj`) which can be replayed with `vd -p
session.vd input.csv` — interactive exploration becomes a versionable,
diffable, reproducible script.

## When to pick it / when not to

Pick `vd` when the question is "*what is even in this file*" — open a
multi-gigabyte CSV from an unfamiliar source, hit `F` on every column
in turn, see the cardinality / distribution / null-rate without
writing a single `pandas.value_counts()`. Pick it when you need to
**inspect five different file formats during one investigation** and
don't want to keep switching between [`xsv`](../xsv/) /
[`miller`](../miller/) / [`jq`](../jq/) / `parquet-tools` / `xlsx2csv`
— `vd file1.csv file2.parquet file3.json file4.xlsx` opens all four
in tabbed sheets with the same keymap. Pick it for **format
conversion** when the source has type quirks a one-line shell pipe
doesn't survive (Excel dates, mixed-type JSON arrays, nested objects
that need flattening with `(` / `)`). The replayable `.vd` command log
makes it useful as a **one-off ETL recorder**: explore interactively,
save the keystroke log, ship the `.vd` file alongside the data and
anyone can re-run the same transform headlessly. Pairs naturally with
[`duckdb`](../duckdb/) (when the answer needs SQL aggregation across
files), [`harlequin`](../harlequin/) (when the data is already in a
warehouse), and [`miller`](../miller/) (for batch / scripted CSV-TSV-
JSON streaming).

Skip it for true big-data work — VisiData loads sheets into memory
(it streams CSV / JSON / parquet readers but materialises the working
view); for billion-row analytics use [`duckdb`](../duckdb/) /
[`datafusion`](../datafusion/). Skip it for non-tabular structured
exploration like deeply-nested JSON without an obvious row axis —
[`jless`](../jless/) / [`fx`](../fx/) are better there. Skip it as
a *production* ETL runtime — a recorded `.vd` script is great for
ad-hoc reproducibility, but dependency-managed pipelines belong in
[`dlt`](../dlt/) / [`kedro`](../kedro/) /
[`prefect`](../prefect/) / [`dagster`](../dagster/). And note the
license: GPL-3.0, copyleft — fine to *use* anywhere (CLI invocation
is not derivative work), but importing the `visidata` Python module
into a closed-source product triggers GPL obligations on that product.

## Example invocations

```bash
# Open one or more files (any supported format)
vd data.csv
vd data.parquet
vd data.json
vd data.xlsx
vd data.csv data.parquet data.json    # tabbed sheets

# Open a directory as a sheet (one row per file)
vd ~/Downloads

# Open a sqlite database (one row per table; Enter to open a table)
vd mydb.sqlite

# Pipe stdin
ps auxf | vd -f tsv

# Non-interactive format conversion
vd data.json -b -o data.csv          # json -> csv, batch mode
vd events.csv -b -o events.parquet   # csv -> parquet

# Replay a saved command log against new input
vd -p clean.vd raw_2026_04.csv -o clean_2026_04.csv

# Useful interactive keystrokes:
#   F       frequency table of the current column
#   [ / ]   sort asc / desc by current column
#   | / \   filter rows: keep / drop matching regex
#   +       aggregate current column (sum/mean/min/max/distinct)
#   =       add a derived column from a Python expression
#   gw      join sheets on selected key columns
#   Ctrl+S  save current sheet (format inferred from extension)
#   Ctrl+D  view command log
#   Ctrl+Y  save command log as .vd / .vdj
#   q       close current sheet, gq quit all

# Verify
vd --version    # saul.pw/VisiData v3.3
```
