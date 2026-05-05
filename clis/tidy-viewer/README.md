# tidy-viewer

> **Pretty-printer for CSV / TSV in the terminal** — a Rust
> CLI (`tv`) that reads delimited data from a file or stdin
> and prints a column-aligned, type-aware, color-coded preview:
> numbers right-aligned with consistent precision, NA values
> dimmed, headers underlined, long strings truncated with an
> ellipsis. Pinned to **v1.8.93** ([LICENSE](https://github.com/alexhallam/tv/blob/main/LICENSE),
> MIT).

Source: <https://github.com/alexhallam/tv>

## TL;DR

`tv data.csv` shows you what `cat data.csv | column -t -s,`
*should* have shown you: a clean, R-tibble-style table preview
with the first ~10 rows, the column headers underlined, the
numeric columns right-aligned to a shared decimal point, the
string columns left-aligned, NA / `null` / empty cells rendered
in a dim color, and a footer telling you total rows × columns.

It's pipeable (`xsv slice -n 100 big.csv | tv`), respects
`NO_COLOR`, supports any single-byte delimiter (`-d $'\t'` for
TSV, `-d '|'` for psv), accepts a custom NA token, lets you
pick the row count (`-n 30`), and has a `-S` switch for the
"give me the structure, not the data" summary that mirrors
`str()` in R or `df.dtypes` in pandas — column name + inferred
type + first few values.

## Why it's interesting

There are two camps in terminal CSV tooling: **manipulation**
(xsv, qsv, miller, csvkit, dasel) and **interactive browsing**
(visidata, tabiew, csvlens, harlequin). `tv` slots into the
gap between them: it's a *non-interactive viewer*, designed to
be the default "what does this file look like" reach inside
shell pipelines and one-liners. No TUI to enter and exit, no
keybindings to remember, no curses redraw — just `tv file.csv`,
the table prints, you're back at the prompt.

The opinionated formatting (right-aligned numerics, dim NAs,
truncated long strings, fixed preview row count) is what
makes it actually *readable* — the same data through `column
-t -s,` is technically aligned but visually unparseable. `tv`
treats the terminal as a typesetting target the way R's
`tibble::print` does, not as a generic monospace dump.

## Install

```bash
# Cargo
cargo install tidy-viewer

# Homebrew
brew tap alexhallam/tap
brew install tidy-viewer

# verify
tv --version    # 1.8.93
```

## Examples

```bash
# default: first ~10 rows, type-aware alignment
tv data.csv

# TSV
tv -d $'\t' data.tsv

# more rows
tv -n 30 data.csv

# structure summary instead of data preview
tv -S data.csv

# treat "missing", "-", and "" as NA, render them dim
tv -a "missing,-," data.csv

# pipe-friendly: works on stdin
xsv slice -n 100 huge.csv | tv

# strip color (or set NO_COLOR=1)
tv -c 0 data.csv

# preview only certain columns by upstream filtering
xsv select name,age,score data.csv | tv
```

## Use when

- You want a one-shot, scrollback-friendly preview of a CSV
  inside a shell pipeline, without entering an interactive
  TUI like visidata / tabiew.
- You're inspecting machine-generated data dumps (model
  predictions, ETL outputs, log exports) and want NAs and
  numeric precision to *look* like NAs and numerics, not
  blend into the strings.
- You're scripting a notebook-style "preview every CSV in
  this dir" pass: `for f in *.csv; do echo "== $f =="; tv $f;
  done`.
- You like R's `tibble` / `print.data.frame` aesthetic and
  want the same thing for `bash`.

Skip `tv` when you need to *edit* / filter / sort / pivot
interactively — reach for visidata or tabiew. Skip when the
file is millions of rows and you want random-access browsing
rather than the head — reach for csvlens. And skip when you
need to actually transform the data — that's xsv / qsv /
miller territory; `tv` is strictly a viewer.
