# xsv

> **A fast CSV command-line toolkit** in Rust — index, slice,
> select, search, sort, join, frequency-count, and stats over
> very large CSV files without loading them into memory.
> Pinned to **0.13.0**
> ([UNLICENSE](https://github.com/BurntSushi/xsv/blob/master/UNLICENSE),
> Unlicense / public-domain).

Source: <https://github.com/BurntSushi/xsv>

Category: data / dev-utility (CSV processing)

## TL;DR

`xsv` is the swiss-army knife for CSV at the shell. Sub-commands
(`headers`, `select`, `search`, `slice`, `sort`, `join`, `stats`,
`frequency`, `fmt`, `fixlengths`, `partition`, `sample`, `index`,
`count`, `flatten`, `cat`) compose like Unix filters. Building
an `.idx` sidecar with `xsv index foo.csv` turns subsequent
`slice` / `sample` / `count` calls into O(1) seeks even on
multi-GB files. The author (BurntSushi, also of `ripgrep`) put
it in maintenance mode in favor of [`qsv`](https://github.com/dathere/qsv),
but the binary remains rock-solid for everyday CSV wrangling.

## Install

```bash
# Homebrew
brew install xsv

# Cargo
cargo install --locked xsv
```

## Why it sits in the zoo

Most "data wrangling" CLIs assume you have pandas / DuckDB /
Polars in the loop. `xsv` is the opposite philosophy: a single
~3 MB binary that streams, indexes, and joins CSV at near-disk
speed with zero runtime dependencies. It is the canonical
"`grep` for tabular data" reference point against which every
later catalog entry (`csview`, `tv`, `qsv`, `vd`) gets compared.
