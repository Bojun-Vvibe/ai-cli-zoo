# tabiew

- **Repo:** https://github.com/shshemi/tabiew
- **Version:** v0.13.0 (2026-03-14)
- **License:** MIT ([LICENSE](https://github.com/shshemi/tabiew/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install tabiew` · `cargo install tabiew` · `.deb` / `.rpm` on the GitHub release page · prebuilt static binaries (`tw-aarch64-apple-darwin`, `tw-x86_64-unknown-linux-musl`) · binary name is `tw`

## What it does

`tabiew` (`tw`) is a TUI table viewer for tabular files — CSV, TSV,
Parquet, JSONL, Apache Arrow, Excel, ODS — backed by the Polars
query engine, so it can open multi-gigabyte files and run real SQL on
them without loading the whole file into Python.

- Opens a file directly (`tw data.parquet`) into a paginated,
  keyboard-driven table view with column-aware horizontal scrolling
  and a status bar showing row count, dtypes, and current selection.
- `:` opens a Polars-SQL command line where you can `SELECT col1,
  col2 FROM df WHERE col3 > 100 ORDER BY col1 LIMIT 50` against the
  loaded frame; results land in a new tab. Multiple files become
  multiple frames you can join across.
- Side-by-side schema view (`S`) shows column name, dtype, null
  count, and per-column min/max/distinct stats — the "what does this
  dataset look like" question answered without opening pandas.
- Fuzzy column-name search, regex row filters, persistent column
  pinning, and an explicit "view a single row vertically" mode for
  wide tables (LLM eval JSONL with 40 columns of metadata is the
  motivating case).
- Saves the current view back out as CSV / Parquet / JSONL via `:write
  <path>`; pipes work too (`cat data.csv | tw -`).

## Why it's interesting for AI / agent workflows

LLM eval datasets, RAG retrieval logs, fine-tune training sets, and
agent trace exports almost always end up as JSONL or Parquet with
dozens of columns. `tabiew` is the lowest-friction way to inspect
them: open the file, type a SQL filter, sanity-check the labels,
spot the duplicate prompts, find the rows where the judge model
disagrees with the human grader. Polars is fast enough that 10M-row
trace files stay interactive on a laptop, which is the regime where
pandas in a notebook starts swapping. Pairs naturally with
[`csvlens`](../csvlens/) (lighter, no SQL) and
[`harlequin`](../harlequin/) (full DuckDB / SQL editor for
multi-file analyses).

## When NOT to use it

Skip it for tiny files where `bat` / `column -t` / `csvlook` is
already enough. Skip it when you need real SQL across many files
with joins and CTEs as the primary use case — [`harlequin`](../harlequin/)
or DuckDB CLI is the better tool. Skip it for streaming data — `tw`
loads (or memory-maps) a file; for a live tail use [`lnav`](../lnav/)
or [`tailspin`](../tailspin/).
