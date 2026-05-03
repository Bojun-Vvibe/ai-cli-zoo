# zstd

> **Zstandard — the modern general-purpose
> lossless compressor from Meta** that hits a
> point on the speed/ratio curve nothing else
> reaches: gzip-class ratios at multi-GB/s
> decompression, with a `--long` window up to 2
> GiB, dictionary mode for tiny-message corpora,
> and a `--adapt` mode that retunes the level to
> match pipe throughput in real time. The
> reference C implementation ships the `zstd`
> CLI, the `libzstd` library, and the `pzstd`
> parallel front-end. Pinned to **v1.5.7**
> ([LICENSE](https://github.com/facebook/zstd/blob/dev/LICENSE),
> BSD-2-Clause; `COPYING` adds a GPL-2.0
> dual-license option for the same code).

Source: <https://github.com/facebook/zstd>

## TL;DR

You have a tarball, a database dump, a log file,
a Docker layer, a Parquet column, an event-bus
message — anything you used to `gzip` or `xz`.
For 90% of those, `zstd` is strictly better:
faster compression at the same ratio, *much*
faster decompression (single-threaded `zstd -d`
is in the 1–2 GB/s range on a modern laptop, gzip
is ~400 MB/s), and a level dial (`-1` to `-22`,
plus `--long` and `--ultra`) that lets you trade
across a far wider envelope than gzip's 1–9 or
xz's 0–9. It is the format Linux kernel images,
btrfs, ZFS, RPM, deb, Arch packages, the Rust
crate registry, and a growing share of the cloud
storage world have already standardized on. It is
not a niche tool — it is the new gzip.

## Install

```bash
# macOS
brew install zstd

# Debian / Ubuntu
sudo apt install zstd

# Build from source (single dependency-free C build)
curl -L https://github.com/facebook/zstd/releases/download/v1.5.7/zstd-1.5.7.tar.gz | tar -xz
cd zstd-1.5.7 && make -j && sudo make install

# Verify
zstd --version    # *** Zstandard CLI (... v1.5.7) ***
```

The CLI is `zstd` (and the convenience wrappers
`unzstd`, `zstdcat`, `zstdmt`); the library is
`libzstd.{so,dylib,a}`; the parallel front-end is
`pzstd`. All ship from the same source tree.

## Use it for

```bash
# 1) Compress a file (default level 3 — fast, good ratio)
zstd big.tar
#   → big.tar.zst, original deleted unless --keep / -k

# 2) Decompress
zstd -d big.tar.zst        # or: unzstd big.tar.zst
#   → big.tar

# 3) Stream — drop-in for `gzip -c` / `gunzip -c`
tar -cf - ./project | zstd -19 --long > project.tar.zst
zstdcat project.tar.zst | tar -xf -

# 4) Multi-threaded compression (one thread per core)
zstd -T0 -19 --long big.bin
#   -T0 = auto-detect cores; --long extends the
#   window to 128 MiB for better ratio on large files

# 5) Maximum-ratio "ultra" mode (slow compression,
#    fast decompression)
zstd --ultra -22 --long firmware.img

# 6) Adaptive level — retune to match pipe throughput
#    (great for `zfs send | zstd | nc`)
zfs send tank/data@snap | zstd --adapt | nc dr-host 9000

# 7) Dictionary training for tiny messages
#    (HTTP responses, JSON events, log lines —
#    ratio jumps from 1.5x to 5x+ with a 64 KiB dict)
zstd --train logs/*.json -o log.dict
zstd -D log.dict -19 today.json
zstd -D log.dict -d today.json.zst

# 8) Verify integrity without writing output
zstd -t archive.zst    # XXH64 checksum check

# 9) Recompress in place
zstd --rm -19 *.log    # delete originals after compress

# 10) Use as a `tar` codec
tar --zstd -cf project.tar.zst project/
tar --zstd -xf project.tar.zst
```

The flag set is large but the muscle-memory
subset is small: `-T0` for parallel, `-19
--long` for "near-best ratio at sane speed",
`--ultra -22 --long` for archive runs, `-d` for
decompress, `--adapt` for pipes.

## Why include it in a CLI catalog

1. **It is the modern default for general-
   purpose lossless compression.** Faster *and*
   smaller than gzip across the board; faster to
   decompress than xz by a wide margin. Any
   pipeline that still does `| gzip -9 |` is
   leaving throughput, ratio, and CPU on the
   floor — `| zstd -T0 -19 --long |` is a free
   upgrade.
2. **Dictionary mode is a category nobody else
   ships well.** For corpora of small messages
   (JSON events, gRPC payloads, HTTP response
   bodies, log lines), training a dictionary
   once and compressing with `-D` regularly hits
   3–5× the ratio of dictionary-less zstd, *and*
   3–10× the ratio of gzip. Nothing in the gzip
   / bzip2 / xz family does this.
3. **The `--adapt` mode is uniquely well-suited
   to streaming.** Send a ZFS snapshot, a
   PostgreSQL `pg_dump`, a `tar` over `nc` — the
   compressor watches the downstream throughput
   and lowers its level if it would otherwise
   stall, raises it on idle bandwidth. No other
   compressor in this space does it.

For an LLM-CLI workflow, `zstd` is the "ship the
context" piece: dump a 200 MB context tree
(`tar --zstd -cf ctx.tar.zst src/`) onto an S3
bucket / artifact store / HTTP endpoint, the
agent or evaluator pulls it back with one
`zstd -d` and a `tar -x`. With `-T0 -19 --long`
you get compression times in the single-digit
seconds and download times bounded by network,
not decompression. The Rust crate registry and
the Linux kernel build already use it for
exactly this reason.

## Vs Already Cataloged

- **Vs [`crabz`](../crabz/):** sibling but
  narrower. `crabz` is a Rust front-end aimed
  primarily at *parallel gzip-compatible*
  compression (`pigz` replacement) and adds zstd
  as a backend. `zstd` is the *reference
  implementation* of the format itself, with
  dictionary training, `--adapt`, `--long`, and
  the canonical `libzstd` library every binding
  in every language wraps. Pick `crabz` if you
  need a `pigz` drop-in that *also* does zstd;
  pick `zstd` if you actually want zstd's full
  surface (dicts, ultra mode, training).
- **Vs [`b3sum`](../b3sum/):** orthogonal —
  `b3sum` is a *hash*, `zstd` is *compression*.
  Both are "modern replacements for an Old
  Standard" (BLAKE3 vs MD5/SHA-1, Zstandard vs
  gzip), but the things they do are
  unrelated.
- **Vs `gzip` / `pigz`:** `zstd` is a strict
  upgrade for any new pipeline. `pigz` remains
  relevant only when compatibility with a
  gzip-only consumer is mandatory (very old
  appliance, hard-coded mime type, etc.).
- **Vs `xz` / `lzma`:** `xz` still wins on
  ratio for very large, highly compressible
  data when *decompression speed doesn't
  matter* and you're willing to pay 10×+ the
  CPU. For everything else (containers, kernel
  images, package managers, HTTP transport),
  `zstd --ultra -22 --long` matches `xz -9` to
  within a few percent and decompresses 5–10×
  faster.
- **Vs `lz4` / `snappy`:** opposite end of the
  curve — those are "barely-compressing,
  CPU-cheap" streaming codecs for hot paths
  (memtable flushes, RPC framing). `zstd -1`
  is in the same ballpark on speed *and* gets
  meaningfully better ratios; `lz4` still wins
  in the absolute lowest-latency case (in-RAM,
  per-message).

## Caveats

- **The format is stable; the CLI flags are
  not perfectly stable across major versions.**
  `--long` window sizes, default dictionary IDs,
  and a few `--ultra`-only flags have shifted.
  For reproducible build artifacts, pin both
  `zstd --version` *and* the exact flag string
  in your build script.
- **`--long=N` requires the *decompressor* to
  allocate a window of size `2^N` bytes.** A
  1.5 GiB archive compressed with `--long=31`
  needs ~2 GiB RAM to decompress. Default
  `--long` (with no argument) is 27 (= 128 MiB)
  which is safe everywhere; raise it
  deliberately.
- **Dictionary mode requires the dictionary to
  be present at decompress time.** Ship it
  alongside the data, or embed its ID in your
  pipeline's metadata; a `.zst` file
  compressed with a missing dict is unusable.
- **`-T0` parallelism is per-frame, not
  per-block.** Tiny inputs see no speedup. Use
  it for ≥ a few MB.
- **BSD-2-Clause + GPL-2.0 dual license**
  ([LICENSE](https://github.com/facebook/zstd/blob/dev/LICENSE),
  [COPYING](https://github.com/facebook/zstd/blob/dev/COPYING))
  — permissive; safe to ship in proprietary
  products with attribution. The library is
  the de-facto-standard implementation behind
  every other zstd consumer in the ecosystem,
  so vendoring it is strongly preferred over
  reimplementing.
