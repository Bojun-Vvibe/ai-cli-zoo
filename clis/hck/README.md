# hck

> **A sharp `cut(1)` clone with regex delimiters,
> output-column reordering, and ripgrep-style
> automatic decompression** — a single ~2 MB Rust
> binary that fills the gap between `cut` and
> `awk`: split on a regex (not a fixed byte),
> reorder output columns, select fields by header
> name or header regex, and let the tool auto-pipe
> through `gzip` / `zstd` / `bzip2` / `xz` / `lz4`
> when the input file extension matches. Pinned to
> **v0.11.5**
> ([LICENSE-MIT](https://github.com/sstadick/hck/blob/master/LICENSE-MIT),
> dual MIT / Unlicense).

Source: <https://github.com/sstadick/hck>

## TL;DR

`hck` ("hack", a rougher form of `cut`) is the
field-extracting tool the Unix toolbox has been
quietly missing for forty years. The original
`cut(1)` is fast and ubiquitous but only splits on
a single literal byte, which means any input
delimited by "one or more spaces" (`ps aux`,
`df -h`, anything aligned for humans) forces a
`tr -s ' '` preprocessing step before `cut` will
behave. `awk` handles the regex-delimiter case but
asks you to learn a small language for what should
be one flag. `xsv` / `qsv` understand RFC 4180 CSV
but are overkill (and slower) when the input is
just whitespace-separated columns. `hck` picks the
narrow niche between them: same flag shape as
`cut` (`-d`, `-f1,3,5-7`), same speed-of-thought
ergonomics, but `-d` accepts a regex by default
(`-d'\s+'` is implicit; `-Ld' '` opts back into
fast literal-byte mode), `-f` reorders output
(`-f3,1,2` writes column 3 first), `-F header` /
`-F` plus `-r` selects by header literal or
header regex, `-e` excludes columns by index, `-E`
excludes by header regex, `-z` auto-decompresses
inputs whose extension matches a known
compressor, and `-Z` gzip-compresses the output
through the multi-threaded `gzp` pipeline. Built
with profile-guided optimisation and SIMD line
splitting, `hck` is consistently faster than `cut`
on single-byte delimiters and faster than `awk` /
`xsv` / `choose` on multi-character / regex
delimiters in the upstream benchmarks.

## Install

```bash
# Homebrew (macOS / Linux)
brew tap sstadick/hck
brew install hck

# Cargo
cargo install hck

# Conda
conda install -c conda-forge hck

# MacPorts
sudo port install hck

# Debian / Ubuntu .deb from releases
curl -LO https://github.com/sstadick/hck/releases/latest/download/hck-linux-amd64.deb
sudo dpkg -i hck-linux-amd64.deb

# Pre-built binary (Linux / macOS / Windows)
# from https://github.com/sstadick/hck/releases

# Maximum-performance build (PGO + native CPU features)
git clone https://github.com/sstadick/hck && cd hck
cargo install just && just install-native

# verify
hck --version    # hck 0.11.5
```

## Basic usage

```bash
# split on the default whitespace regex (\s+) and
# pick columns 1, 8, 19 from `ps aux` output
ps aux | hck -f1,8,19 | head

# explicit single-byte delimiter — fast path,
# behaves like cut -d',' -f1,8,19
hck -Ld, -f1,8,19 ./hyper_data.csv

# reorder output columns (cut cannot do this)
ps aux | hck -f2,1,3- | head

# regex delimiter — split on $;$ literally
printf 'a$;$b$;$c\n' | hck -Ld'$;$' -f2

# select columns by header literal
hck -d, -F 'name' -F 'email' ./users.csv

# select columns by header regex
ps aux | hck -r -F '^ST.*' -F '^USER$' | head

# exclude columns by header regex
ps aux | hck -r -E 'CPU' -E '^ST.*' | head

# change output record separator (D, capital)
ps aux | hck -D'___' -f2,1,3 | head

# auto-decompress gz / bz2 / xz / zst input
hck -z -f1-3 ./access.log.gz

# multi-threaded gzip-compress the output
hck -Z -f1,3 ./big.tsv > big.subset.tsv.gz

# config? none — flag-driven, --help is the
# canonical reference and stays under two screens
```

## When to choose

- **You are doing field extraction in a shell
  pipeline and `cut`'s "split on one literal byte"
  is the friction** — `hck` accepts a regex by
  default, so `ps aux | hck -f1,3,5` and
  `df -h | hck -f5,1` *just work* with no
  `tr -s ' '` preamble. That alone is the reason to
  install it.
- **You need to reorder output columns** — `cut`
  always emits in input order; `hck -f3,1,2` emits
  in flag order. This is what people reach for `awk
  '{print $3, $1, $2}'` for, but with no language
  to learn.
- **You select fields by header name, not just
  index** — `hck -d, -F email -F id < users.csv`
  picks the columns whose header literals match.
  With `-r`, headers can be regex
  (`-F '^total_.*'`).
- **Your inputs are sometimes compressed** — `hck
  -z` recognises `.gz` / `.bz2` / `.xz` / `.zst` /
  `.lz4` / `.br` / `.lzma` / `.Z` extensions and
  shells out to the right decompressor (the same
  approach as `ripgrep -z`), so log analysis on a
  mixed tree of plain + rotated files stops needing
  `for f in *; do zcat "$f" | hck …`.
- **Throughput on big files matters** — on the
  upstream 7 M-line benchmark, `hck -Ld,` is the
  fastest of the popular field tools at the
  single-byte-delimiter path and competitive with
  `tsv-utils` on multi-character delimiters; both
  are several times faster than `cut` and `awk`.

## When NOT to choose

- **Your input is RFC 4180 CSV with quoted commas
  inside fields** — `hck` is not a CSV parser; it
  splits on the delimiter wherever it appears, just
  like `cut`. Use [`xsv`](../xsv/) /
  [`qsv`](../qsv/) /
  [`miller`](../miller/) /
  [`tidy-viewer`](../tidy-viewer/) /
  [`csvq`](../csvq/) when quoting matters.
- **You need row filtering or arithmetic** — `hck`
  is field selection only; `awk` /
  [`frawk`](https://github.com/ezrosent/frawk) /
  `mlr` (Miller) are the right tools for "print
  the row when column 3 > 1000" or running totals.
- **You work in CSV-with-headers + need explicit
  type-aware operations** — Miller / `qsv` give you
  `cat`, `cut`, `tac`, `sort`, `stats` with full
  CSV semantics in one tool; `hck` is a sharp knife
  for one cut.
- **Your delimiter spans newlines** — `hck` is a
  line-oriented tool; embedded newlines in fields
  will not be respected.

## Why it fits the zoo

The zoo's text-shredding cluster
([`choose`](../choose/),
[`miller`](../miller/),
[`qsv`](../qsv/),
[`xsv`](../xsv/),
[`dasel`](../dasel/),
[`jc`](../jc/),
[`tidy-viewer`](../tidy-viewer/)) collects the
column-and-record tools people actually pipe `ps`
/ `df` / log lines through. `hck` slots in next to
[`choose`](../choose/) as the
"`cut` but better" half of the cluster: same
single-purpose shape, same flag spelling, same
sub-second startup, just a regex-aware delimiter,
output-column reordering, and ripgrep-style
auto-decompression bolted on. It pairs naturally
with `ripgrep` / `bat` for the
"pre-filter-then-extract-then-pretty-print"
pipeline that LLM-CLI front-ends and dashboard
scripts both lean on.

## Upstream pointers

- Repo: <https://github.com/sstadick/hck>
- Release notes: <https://github.com/sstadick/hck/releases>
- Licenses: dual
  [MIT](https://github.com/sstadick/hck/blob/master/LICENSE-MIT)
  / [Unlicense](https://github.com/sstadick/hck/blob/master/UNLICENSE)
- Changelog: <https://github.com/sstadick/hck/blob/master/CHANGELOG.md>
- Author: [@sstadick](https://github.com/sstadick)
  (also [`crabz`](../crabz/), a multi-threaded
  pigz-like compressor, and `gzp`, the underlying
  multi-threaded compression library `hck`'s `-Z`
  flag uses).
