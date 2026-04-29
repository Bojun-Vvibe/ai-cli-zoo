# crabz

> **A multi-threaded gzip / zlib / BGZF / Mgzip /
> deflate / snap compressor in one Rust binary —
> like `pigz`, but cross-platform, with more output
> formats and pluggable compression backends
> (`zlib-ng`, `libdeflate`, pure Rust)** — drop-in
> replacement for `pigz` that runs natively on
> Linux, macOS, and Windows, supports BGZF
> (bgzip-compatible block-gzip used by `tabix` /
> `samtools`) and Mgzip out of the box, and lets
> you pin compression threads to specific CPU cores
> for predictable throughput when running multiple
> instances. Pinned to **v0.10.0**
> ([LICENSE-MIT](https://github.com/sstadick/crabz/blob/main/LICENSE-MIT),
> dual MIT / Unlicense).

Source: <https://github.com/sstadick/crabz>

## TL;DR

`crabz` is the modern Rust reimplementation of
`pigz` with three things going for it: it works
unmodified on Windows (pigz is fundamentally a
Unix tool), it understands more output formats
(gzip, zlib, raw deflate, BGZF, Mgzip, snap — not
just gzip), and it lets you pick the deflate
backend at install time (`flate2 + zlib-ng` for
peak throughput, `flate2 + miniz_oxide` for
pure-Rust portability, `libdeflate` for the
fixed-block formats). The CLI surface is
deliberately small: read from stdin or `<FILE>`,
write to stdout or `-o output.gz`, set
compression level with `-l 1..=9`, set thread
count with `-p N`, optionally pin the worker pool
to physical cores starting at index `-P 4`, switch
format with `-f bgzf` / `-f mgzip` / `-f snap`,
and decompress with `-d`. The single
non-obvious-but-useful flag is `--in-place`, which
compresses (or decompresses) the input file in
situ and removes the source on success — the
ergonomic `pigz file.txt → file.txt.gz` flow
without the implicit-source-removal magic. On the
upstream benchmark (Ubuntu, Ryzen 9 3950X, 64 GB
RAM, 7 M-line dataset) `crabz` with `zlib-ng` is
30–50 % faster than `pigz` at level 6, and decode
is ~25 % faster across the board.

## Install

```bash
# Homebrew (macOS / Linux)
brew tap sstadick/crabz
brew install crabz

# Cargo (defaults to flate2 + zlib-ng on supported targets)
cargo install crabz

# Pure-Rust build (Windows-friendly, no system zlib)
cargo install crabz --no-default-features \
                   --features deflate_rust

# libdeflate backend (fastest for BGZF / Mgzip blocks)
cargo install crabz --no-default-features \
                   --features libdeflate

# Conda
conda install -c conda-forge crabz

# Debian / Ubuntu .deb from releases
curl -LO https://github.com/sstadick/crabz/releases/latest/download/crabz-linux-amd64.deb
sudo dpkg -i crabz-linux-amd64.deb

# verify
crabz --version    # crabz 0.10.0
```

## Basic usage

```bash
# default: gzip-compress stdin to stdout, level 6, all cores
crabz < big.txt > big.txt.gz

# compress a file in place — removes big.txt on success
crabz --in-place big.txt        # → big.txt.gz

# decompress in place — removes big.txt.gz on success
crabz --decompress --in-place big.txt.gz   # → big.txt

# pick a compression level (1 fastest, 9 smallest)
crabz -l 9 < big.txt > big.txt.gz

# limit thread count (default = all logical cores)
crabz -p 8 -l 6 < big.txt > big.txt.gz

# pin compression threads to physical cores 4..7
# (lets two instances of `crabz -p 4 -P 0` and
#  `crabz -p 4 -P 4` coexist without stealing cores)
crabz -p 4 -P 4 < big.txt > big.txt.gz

# BGZF — bgzip-compatible block gzip, indexable by tabix
crabz -f bgzf < ref.fasta > ref.fasta.gz
tabix -p fasta ref.fasta.gz   # works

# Mgzip — multi-block gzip, parallel-decompressible
crabz -f mgzip < big.txt > big.txt.mgz
crabz -d -f mgzip -p 8 < big.txt.mgz > big.txt

# Snap (Snappy framing format) for low-CPU / high-RAM
crabz -f snap < big.txt > big.txt.sz

# Pipe-friendly — drops cleanly into a tar pipeline
tar -cf - ./project | crabz -p 16 -l 6 > project.tar.gz
```

## When to choose

- **You want `pigz` performance on Windows or in a
  pure-Rust toolchain** — `pigz` does not have a
  first-class Windows build; `crabz`'s pure-Rust
  feature does, with the same parallel-deflate
  speedup. Cross-compilation to musl / Windows /
  macOS-arm64 is one `cargo build --target=...`.
- **Your output format is BGZF or Mgzip** — both
  are block-gzip variants used in
  bioinformatics (`tabix` indexing) and
  multi-block-decompress workflows; `pigz` only
  emits stream-gzip. `crabz -f bgzf` /
  `crabz -f mgzip` is the single-binary answer.
- **You want a faster gzip than `pigz`** — at level
  6 with the `zlib-ng` backend, `crabz` is 30–50 %
  faster than `pigz` on the upstream benchmark, and
  decompression is ~25 % faster across all levels.
  The library underneath ([`gzp`](https://github.com/sstadick/gzp))
  is a Rust crate you can also use directly.
- **You run multiple compressions on the same
  host** — `-P` thread pinning lets you cleanly
  partition cores between concurrent `crabz`
  invocations without thread-storm contention.
- **You need format flexibility in CI** — same
  binary handles gzip, zlib, raw deflate, BGZF,
  Mgzip, and Snappy framing. One install, no
  per-format tool.

## When NOT to choose

- **You need zstd / xz / brotli / bzip2 output** —
  `crabz` is the deflate-and-friends family only
  (gzip / zlib / deflate / BGZF / Mgzip + Snappy
  framing). For zstd reach for `zstd` (multi-thread
  with `-T0`); for xz, `pixz` / `xz -T`; for
  brotli, `brotli`. They are out of scope by
  design.
- **You only ever use gzip-stream and pigz is
  already installed everywhere you work** — `pigz`
  is fine; `crabz` is the right pick when you want
  Windows, BGZF, or the speed delta.
- **You want a fast single-threaded gzip** — single
  thread is `crabz`'s least-favourable case. For
  small files the constant overhead of the
  thread-pool dwarfs the work; just use `gzip` /
  `pigz -p 1` / `gzp`-as-library.
- **You need split / merge / random access into
  gzip streams** — that is `bgzip` + `tabix` /
  `htslib` territory. `crabz` produces BGZF
  output, but the reading-and-indexing tooling
  lives elsewhere.

## Why it fits the zoo

The zoo's archive-and-compression cluster
([`ouch`](../ouch/),
[`hexyl`](../hexyl/),
[`pigz`](https://zlib.net/pigz/) (not in zoo),
[`zstd`](https://github.com/facebook/zstd) (not in
zoo)) collects single-binary, drop-in successors to
the classic Unix compression tools that people
actually keep in `$PATH`. `crabz` slots in as the
parallel-deflate-and-friends entry: it is to
`gzip` / `pigz` what [`havn`](../havn/) is to
`nmap` — same shape, fewer surprises, single static
binary, sane defaults. It pairs with
[`hck`](../hck/) (same author, the `-Z` flag of
which is built on the same `gzp` library) and with
[`ouch`](../ouch/) (which dispatches to gzip /
zstd / xz / bzip2 by extension; `crabz` is what
you reach for when *gzip specifically* is
load-bearing on a hot path).

## Upstream pointers

- Repo: <https://github.com/sstadick/crabz>
- Release notes: <https://github.com/sstadick/crabz/releases>
- Licenses: dual
  [MIT](https://github.com/sstadick/crabz/blob/main/LICENSE-MIT)
  / [Unlicense](https://github.com/sstadick/crabz/blob/main/UNLICENSE)
- Changelog: <https://github.com/sstadick/crabz/blob/main/CHANGELOG.md>
- Underlying library:
  [`gzp`](https://github.com/sstadick/gzp) —
  multi-threaded compression / decompression as a
  Rust crate, used both by `crabz` itself and by
  [`hck`](../hck/)'s `-Z` output mode.
- Author: [@sstadick](https://github.com/sstadick)
  (also [`hck`](../hck/), the regex-delimiter
  `cut(1)` clone in this catalog).
