# csvlens

- **Repo:** https://github.com/YS-L/csvlens
- **Version:** v0.12.1 (latest stable, 2025)
- **License:** MIT ([LICENSE](https://github.com/YS-L/csvlens/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install csvlens` · `cargo install --locked csvlens` · binary releases on the GitHub release page · also packaged in nixpkgs and AUR

## What it does

`csvlens` is a terminal CSV viewer that treats a CSV / TSV / arbitrary-delimited
file the way `less` treats a text file: open it, scroll it, search it, exit.
The difference is that it actually parses the rows, so columns line up, the
header row is sticky while you scroll, and you can jump to a column by name
(`>name`) instead of counting commas. It streams large files lazily, so opening
a multi-gigabyte CSV is instant — only the on-screen window is parsed, not the
whole file. Filter mode (`/pattern`) hides rows that do not match a regex,
fuzzy-find mode (`F`) jumps you to the first match, and `Find` mode highlights
all hits in the current view. Column freezing (`f`), column hiding (`-`), and
column reordering keep wide schemas readable. The status bar always shows
`row x of y, col i of j` so you know where you are. It reads from stdin
(`cat foo.csv | csvlens`), supports custom delimiters (`-d ';'`, `-d $'\t'`),
and can preview Parquet via the `--parquet` flag (delegates to a small
in-process reader, no external dependency). Editing is intentionally not
supported — it is a viewer, not a spreadsheet — which keeps the binary tiny
(~3 MB static) and the keymap small enough to learn in five minutes.

## When to pick it / when not to

Pick `csvlens` whenever a CSV is too wide for `less -S` to be useful and too
big to load into a notebook just to peek. The classic case: production data
dump arrives, you want to know "does column `email` ever contain a comma" or
"what are the distinct values in `region`" before writing a single line of
pandas. `csvlens foo.csv`, `>region`, `/asia`, scroll, done — no Python
process, no editor reflow, no awk that explodes on quoted fields. It pairs
naturally with [`miller`](../miller/) (mlr does the transform; csvlens shows
you what came out), [`dasel`](../dasel/) (when the CSV is one branch of a
larger config tree), and [`jless`](../jless/) (the JSON cousin — same
sticky-header viewer model). Use it as the post-pipeline previewer for any
shell that produces tabular output: `psql -c "..." --csv | csvlens`,
`duckdb -csv -c "select * from read_parquet(...)" | csvlens`, or
`aws s3 cp s3://bucket/big.csv - | csvlens`.

Skip it when you need to **edit** the file — use a real spreadsheet, `vd`
(visidata), or a notebook. Skip it for **aggregations or joins** — that is
`miller`, `qsv`, `duckdb`, or `xsv`'s job; csvlens does not know how to
group, sort by computed key, or join. Skip it for **non-tabular CSV-shaped
files** like CSV with embedded newlines in unquoted fields or BOM-laden
exports from old Excel exports — csvlens uses a strict RFC-4180 parser and
will report a parse error rather than guess. Skip it on truly enormous files
where you need columnar pruning at read time — convert to Parquet first and
either use `csvlens --parquet` or query with `duckdb`. And skip it if you
want a persistent dashboard — csvlens is a one-shot interactive viewer, not
a refresh-on-interval TUI like [`viddy`](../viddy/) or
[`toolong`](../toolong/).

## Example invocations

```bash
# Open a CSV with a sticky header and column alignment
csvlens orders.csv

# TSV from a database dump
psql -At -F$'\t' -c 'select * from users' mydb | csvlens -d $'\t'

# Custom delimiter (semicolon-separated European export)
csvlens -d ';' export.csv

# Jump to a column by name once open: type ">email" then Enter
# Filter rows by regex: type "/2024-1[12]" then Enter
# Find (highlight all matches): type "F" then a pattern

# Preview a Parquet file without booting a notebook
csvlens --parquet events.parquet

# Pipe in from a query engine
duckdb -csv -c "select region, count(*) from read_parquet('s3://b/*.parquet') group by 1" \
  | csvlens

# Combine with miller for transform-then-view
mlr --csv filter '$status == "FAIL"' then sort -f region orders.csv \
  | csvlens

# Open with a specific row width cap (truncate long cells)
csvlens --columns-width 40 wide_schema.csv
```
