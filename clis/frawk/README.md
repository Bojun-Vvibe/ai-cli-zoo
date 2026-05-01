# frawk

## What it does
A **JIT-compiled `awk` implementation** in Rust that parses a
substantial subset of POSIX `awk` + `gawk` extensions and compiles each
program to either Cranelift or LLVM machine code at startup, then
streams input through a parallel chunked CSV / TSV / regex-split parser
that uses SIMD (SSE2 / AVX2) for delimiter scanning. Adds first-class
`-i csv` / `-i tsv` input modes that handle quoted fields and embedded
newlines correctly (a long-standing footgun for `awk -F,` on real CSV),
matching `-o csv` / `-o tsv` output modes that re-quote on write,
parallel execution across input chunks for embarrassingly-parallel
record-at-a-time scripts (`frawk -pr` runs the per-record block on N
threads and merges per-chunk `END` aggregates), and integer / float /
string typing with explicit `int(x)` / `hex(x)` coercions instead of
`awk`'s implicit-but-surprising type promotion. Single static binary,
no runtime, drop-in for the `mawk` / `gawk` / `nawk` script shape on
inputs measured in gigabytes.

## Why it's interesting
Different shape from `mawk` (single-threaded bytecode VM, no real CSV,
no SIMD), from `gawk` (reference implementation, single-threaded, no
real CSV without `-l csv` extension, slower than `mawk` on most loads),
from `awk` POSIX (no associative-array assignment as expression, no
gensub, no real CSV), from `mlr` (verb-pipeline ergonomics over named
columns — a different mental model: `mlr put '$x=$y+$z'` vs frawk's
`{ print $1+$2 }`; pick mlr when columns have names and the workload is
"compose verbs", pick frawk when you're already thinking in awk and
want it 5-20× faster on multi-GB CSV / TSV), from `qsv` (CSV-only,
verb-style, not a programming language — pick qsv for `select` /
`join` / `sort` / `dedup` with sane defaults, frawk when the per-record
logic is a script). frawk is the *fast parallel awk for big tabular
text* shape: pick it specifically when you have a multi-gigabyte CSV /
TSV / log file, an existing `awk`-shaped script, and the bottleneck is
the awk runtime not the disk. Do **not** pick it for the last 5% of
gawk's surface (gensub edge cases, network I/O, `mktime`), for non-
tabular structured data (use `jq` / `dasel` / `mlr` for JSON), or when
the dataset already fits in DuckDB / pandas (relational queries beat
record-at-a-time).

## Niche category
JIT-compiled parallel awk with SIMD CSV/TSV — drop-in awk for multi-GB
tabular text where mawk / gawk are the bottleneck.

## Repo
https://github.com/ezrosent/frawk

## Version pinned
`v0.4.7` (latest tagged release, published 2023-01-02)

## License
- SPDX: `Apache-2.0` (also offered under `MIT` — dual-licensed)
- License file in upstream repo: `LICENSE-APACHE` (companion: `LICENSE-MIT`)

## Install
```sh
# Cargo (any platform; default backend is Cranelift)
cargo install frawk

# With LLVM backend for fastest steady-state throughput on long-running
# scripts (requires LLVM 12 dev headers on the build host)
cargo install frawk --features llvm_backend

# Nix
nix-env -iA nixpkgs.frawk

# Prebuilt binaries
# https://github.com/ezrosent/frawk/releases/tag/v0.4.7
```

## Usage examples
```sh
# Drop-in awk: sum column 2 of a 12 GB TSV
frawk '{ s += $2 } END { print s }' big.tsv

# Real CSV mode (handles quoted fields + embedded newlines correctly)
frawk -i csv '$3 == "US" { c++ } END { print c }' orders.csv

# Parallel execution: run the per-record block on 8 threads, merge END
frawk -pr -j 8 -i csv \
  '{ rev[$3] += $5 + 0 } END { for (k in rev) print k, rev[k] }' \
  sales-2026.csv

# Convert TSV → CSV with quoting handled on output
frawk -i tsv -o csv '{ print $1, $2, $3 }' events.tsv > events.csv

# Group-by with explicit numeric coercion (frawk is typed; awk's
# implicit string-vs-number is the usual surprise)
frawk -i csv '{ t[$2] += int($4) } END { for (k in t) print k, t[k] }' \
  metrics.csv

# Dump the parsed AST / IR for debugging a hot script
frawk --dump-cfg '{ print $1 }' /dev/null
```
