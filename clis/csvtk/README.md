# csvtk

> **A cross-platform, fast, practical CSV/TSV toolkit** — a single
> Go binary giving you ~30 subcommands (`cut`, `filter2`, `freq`,
> `join`, `mutate2`, `pretty`, `summary`, `plot`, `csv2json`, ...)
> that operate on delimited text streams without ever needing
> SQLite, pandas, or a spreadsheet. Pinned to **v0.37.0**
> ([LICENSE](https://github.com/shenwei356/csvtk/blob/master/LICENSE),
> MIT).

Source: <https://github.com/shenwei356/csvtk>

## TL;DR

`csvtk` is what you reach for when you have a 2 GB TSV and want
to answer "what's the median of column 4 grouped by column 7?"
without writing any code. It handles CSV, TSV, and (since
v0.37.0) LZ4-compressed inputs natively, treats the header row
as named columns by default (`-f sample_id,depth` instead of
`-f 1,4`), and pipes cleanly between subcommands so a typical
session looks like a small Unix pipeline:
`csvtk grep -f gene -p BRCA1 in.tsv | csvtk cut -f sample,vaf | csvtk summary -g sample -f vaf:mean`.
The author maintains it primarily for bioinformatics workloads
(it ships alongside `seqkit`), but nothing in the tool is
domain-specific — the same commands work on web logs, billing
exports, or survey CSVs. Output is plain delimited text by
default; `csvtk pretty` renders Markdown / box-drawing tables for
terminals and `csvtk plot hist|box|line` writes PNG/SVG/PDF.

## Install

```bash
# Homebrew (macOS / Linux)
brew install csvtk

# Conda
conda install -c bioconda csvtk

# Go
go install github.com/shenwei356/csvtk/csvtk@latest

# Pre-built binary (any OS, no toolchain needed)
curl -LO https://github.com/shenwei356/csvtk/releases/download/v0.37.0/csvtk_darwin_arm64.tar.gz
tar xf csvtk_darwin_arm64.tar.gz
sudo install csvtk /usr/local/bin/

# Verify
csvtk version    # csvtk v0.37.0

# Optional: shell completion
csvtk genautocomplete --shell zsh > ~/.zsh/completion/_csvtk
```

## Example usage

```bash
# 1. Pretty-print a TSV with the new "regular" style (v0.37.0)
csvtk pretty -t -s regular results.tsv

# 2. Filter rows where column `vaf` > 0.05, then keep three columns
csvtk filter2 -t -f '$vaf > 0.05' calls.tsv \
  | csvtk cut -t -f sample,gene,vaf

# 3. Group-by + summary (median + count) without leaving the shell
csvtk summary -t -g sample -f vaf:median,vaf:count calls.tsv

# 4. Frequency table of a categorical column, sorted descending
csvtk freq -t -f gene -nr calls.tsv | csvtk pretty -t

# 5. Join two TSVs on a shared column, like SQL inner join
csvtk join -t -f sample_id calls.tsv metadata.tsv > merged.tsv

# 6. Convert TSV ⇄ JSON for downstream tools
csvtk csv2json -t --indent 2 merged.tsv > merged.json
csvtk json2csv -t merged.json | head

# 7. Quick histogram to PNG (requires gnuplot? no — pure Go)
csvtk plot hist -t -f vaf calls.tsv > vaf-hist.png

# 8. Read an LZ4-compressed input directly (new in v0.37.0)
csvtk stats -t huge.tsv.lz4
```

## Why this lives in the zoo

Most "CSV one-liner" tools force a choice between speed
(`xsv`/`qsv` — fast, but limited operations) and expressiveness
(`pandas` — every operation, but you wrote a Python script).
`csvtk` lands in the middle: the subcommand surface is wide
enough to cover ~90% of ad-hoc data wrangling without ever
opening a Python REPL, and column-name addressing means your
pipeline survives upstream column reorders. The v0.37.0 LZ4
support also makes it one of the few CLI table tools that can
read a compressed stream end-to-end without an external
decompressor in the pipe.
