# csview

> **A high-performance CSV / TSV pretty-printer
> for the terminal** — render delimited data as
> a Unicode table with aligned columns, header
> row, frozen left columns, and Tabulate-quality
> output, while staying fast enough to stream
> hundred-megabyte files. Pinned to **v1.3.1**
> ([LICENSE](https://github.com/wfxr/csview/blob/main/LICENSE),
> MIT).

Source: <https://github.com/wfxr/csview>

## TL;DR

`csview` is a single static Rust binary that
reads CSV / TSV / arbitrary-delimited input
from a file or stdin and prints a properly
aligned Unicode-box table to the terminal. It
sits in the same niche as `column -t -s,`,
`csvlook`, and `xsv table` but pushes harder
on three axes: **(1) speed** — built on the
`csv` Rust crate with explicit row buffering,
so it streams a 200 MB CSV in seconds without
loading it all into RAM; **(2) presentation**
— five built-in border styles (`sharp`,
`rounded`, `reinforced`, `markdown`, `ascii`)
with optional column-wise truncation, header
underline, and per-column alignment inferred
from value type; **(3) ergonomics on real files**
— `--delimiter` accepts tab / pipe / semicolon
literally, `--no-headers` flips the first row
back to data, `--indices` selects columns by
1-based index or range, and `--style markdown`
emits a copy-pasteable `| col | col |` table for
notes and PRs. Output is colour-aware via
`isatty`, so piping to a pager or file produces
clean ASCII without escape codes.

## Install

```bash
# Homebrew
brew install csview

# Cargo
cargo install csview

# Arch Linux (AUR)
paru -S csview

# prebuilt binaries
# https://github.com/wfxr/csview/releases

# verify
csview --version    # csview 1.3.1
```

## Basic usage

```bash
# basic table from a CSV
csview data.csv

# TSV from stdin
psql -At -F$'\t' -c 'select * from users limit 20' | csview -d '\t'

# pick columns by index, with rounded borders
csview --indices 1,3,5-7 --style rounded data.csv

# headerless input
csview --no-headers data.csv

# markdown for a PR description
csview --style markdown data.csv | pbcopy

# combine with a CSV toolbelt
qsv search -s status '^5' access.csv \
  | qsv select 'time,status,bytes,uri' \
  | csview --style sharp
```

## When to choose

- **You read CSV / TSV in the terminal daily**
  and want it to stop looking like one giant
  comma-mashed line — a single static binary,
  no Python deps, fast enough to stream big
  files.
- **You want a presentation layer on top of
  [`qsv`](../qsv/) / [`xsv`](../xsv/) / [`miller`](../miller/)
  pipelines** — those tools transform; csview
  renders. The combination is "select + filter +
  pretty-print" in one shell pipe.
- **You paste tables into Markdown documents**
  — `--style markdown` emits exactly the shape
  GitHub / GitLab render, with no manual column
  alignment.

## When NOT to choose

- **You need to transform, not display** — use
  [`qsv`](../qsv/), [`miller`](../miller/),
  [`xsv`](../xsv/), or [`duckdb`](../duckdb/);
  csview has no filtering / aggregation /
  reshaping verbs by design.
- **Your CSV has truly weird quoting** — csview
  uses the Rust `csv` crate's strict reader; if
  you have Excel-exported files with embedded
  newlines and inconsistent quoting, normalise
  with [`qsv input`](../qsv/) first.
- **You want spreadsheet semantics in a TUI** —
  use [`visidata`](../visidata/) or
  [`tabiew`](../tabiew/); csview is one-shot
  print, not interactive.

## Why it fits the zoo

Pairs cleanly with the data-CLI cluster
([`qsv`](../qsv/), [`miller`](../miller/),
[`tabiew`](../tabiew/), [`visidata`](../visidata/),
[`duckdb`](../duckdb/)) and the "make terminal
output legible" cluster ([`bat`](../bat/),
[`delta`](../delta/), [`tabiew`](../tabiew/)).
csview is the smallest possible bridge between
"I have a CSV" and "I can read a CSV in my
terminal" — one binary, no config file, sane
defaults.

## Upstream pointers

- Repo: <https://github.com/wfxr/csview>
- Release notes: <https://github.com/wfxr/csview/releases>
- License: [MIT](https://github.com/wfxr/csview/blob/main/LICENSE)
- Maintainer: [@wfxr](https://github.com/wfxr)
