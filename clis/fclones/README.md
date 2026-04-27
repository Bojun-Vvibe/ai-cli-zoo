# fclones

> **A fast, parallel duplicate-file finder and remover for the
> command line** — walks one or more roots, groups files by
> length, prefilters with a small partial-content hash, then
> confirms with a full BLAKE3 hash to identify byte-identical
> duplicates across millions of files; emits machine-readable
> reports (CSV / JSON / FDUPES) that a separate `fclones group
> | fclones remove` / `link` / `move` / `dedupe` pipeline
> consumes to act on the matches without re-scanning. Pinned to
> **0.35.0** (commit
> `a74f90d293e05856d19a4c0ac2b29b46ef16cf23`,
> [LICENSE](https://github.com/pkolaczk/fclones/blob/main/LICENSE),
> MIT).

Source: <https://github.com/pkolaczk/fclones>

## TL;DR

`fclones` is the duplicate-file finder you reach for when
`fdupes` or `rmlint` is too slow on the corpus you actually
have — a 2 TB photo archive, a `node_modules` graveyard across
twenty client checkouts, a music library with FLAC + transcoded
MP3 copies of the same album. The scan is pipelined and
parallel: the walker streams paths into a worker pool, files
get bucketed by exact length first (no I/O beyond `stat`),
then each bucket is partial-hashed (read the first ~4 KB, hash
with BLAKE3) to discard non-duplicates cheaply, and only the
remaining candidate groups get a full-file BLAKE3 to confirm
byte equality. On NVMe SSDs this saturates the drive; on
spinning disks `--threads main=1` serializes the walker so
seeks stay sequential. The output of `fclones group` is a
stable, deterministic report (`txt` / `csv` / `json` / `fdupes`
formats) that you pipe into `fclones remove` / `link --soft`
/ `link --hard` / `move <dir>` / `dedupe` (reflink on
APFS / Btrfs / XFS) to act on it. The split between *finding*
and *acting* is deliberate: you can review the report, hand-edit
the priority order with `--priority`, then re-feed it to the
action subcommand, instead of trusting a single
`--delete-now` flag. Cross-platform single static binary
(macOS / Linux / Windows), no runtime, no daemon, configured
entirely via flags or `fclones.toml`.

## When to choose

- **You have a corpus too large for `fdupes` / `rmlint`** —
  hundreds of GB to multiple TB of files, where the difference
  between "MD5 every file" and "length-bucket → partial-hash →
  full-hash with BLAKE3" is the difference between "overnight"
  and "an hour". `fclones` is consistently the fastest in
  published benchmarks on both SSD and HDD because the
  prefilter cuts I/O by orders of magnitude before any full
  hash runs.
- **You want find/act decoupled** — `fclones group > dups.txt`
  produces a report you can review, edit (re-order with
  `--priority newest`, exclude paths, drop groups), version-
  control, and then feed to `fclones remove --dry-run <
  dups.txt` for a preview before the destructive run. Tools
  that combine "find and delete" in one shot (`rmlint
  --paranoid`, `fdupes -d`) are scarier on a 2 TB archive.
- **You need reflink-based dedup on APFS / Btrfs / XFS** —
  `fclones dedupe` replaces duplicate file content with
  copy-on-write reflinks, reclaiming disk space without
  changing inodes or breaking `mtime`. This is the right
  primitive for a photo library or a `node_modules` farm
  where you want every checkout to keep its own paths but
  share the underlying blocks. `jdupes` / `fdupes` only
  hardlink (which collapses inodes and changes
  mtime-on-change semantics).
- **You want a stable JSON report for downstream automation**
  — `fclones group -f json` emits an array of groups with
  size, hash, and member paths in a documented schema; pipe
  to `jq` to drive a custom retention policy ("keep newest
  in `~/Pictures/incoming`, prefer files under
  `~/Pictures/sorted/` otherwise") that the built-in
  `--priority` flag does not express.
- **You want a single static binary** with no Python / Perl
  runtime — `cargo install fclones`, or grab the GitHub
  release tarball, or `brew install fclones`. Fits in a
  rescue USB next to `ddrescue` and `restic`.

## Comparable CLIs already in zoo

- **Vs [`dust`](../dust/) / [`duf`](../duf/):** those are
  *disk-usage visualizers* — they tell you which directory is
  big. `fclones` is a *duplicate finder* — it tells you which
  files are byte-identical so you can free space without
  losing data. The natural pipeline is `dust` to find the
  fat directory, then `fclones group <that-dir>` to find the
  duplicates inside it.
- **Vs [`fd`](../fd/):** `fd` is a `find` replacement — it
  enumerates paths matching a name / extension pattern and
  exits. `fclones` enumerates paths *and* groups them by
  content equality. You can compose them: `fd -e jpg . |
  fclones group --stdin` runs the duplicate detection only
  over `*.jpg` rather than the whole tree.
- **Vs [`broot`](../broot/):** `broot` is a tree navigator
  with size aggregation in a TUI; `fclones` is a batch
  duplicate processor with no TUI. They overlap on
  "understand the shape of a big directory" but solve
  different problems — `broot` for browsing, `fclones` for
  reclamation.
- **Vs `rmlint` / `fdupes` / `jdupes`** (not in zoo): these
  are the predecessors. `rmlint` has the broadest feature
  set (also finds empty dirs, broken symlinks, bad UIDs);
  `fdupes` is the C original; `jdupes` is the maintained
  fork with hardlink/reflink support. `fclones` is faster
  than all three on large corpora and has the cleanest
  find/act separation; if you only have a few GB to scan,
  any of them work and `fdupes` is in every distro.

## Caveats

- **`fclones dedupe` requires reflink-capable filesystems**
  (APFS on macOS, Btrfs / XFS on Linux with `reflink=1`,
  ReFS on Windows). On ext4 / HFS+ it will print a clear
  error rather than silently fall back; use `link --hard` if
  you actually want hardlinks (and accept the inode collapse
  semantics).
- **The default hash is BLAKE3, not a cryptographic SHA-2.**
  This is correct for duplicate detection — BLAKE3 is faster
  and the collision probability is negligible at any
  realistic scale — but if you are using `fclones` output
  as evidence of file identity for compliance / forensics,
  pass `--hash-fn sha256` so the report matches what an
  auditor will recompute with `shasum -a 256`.
- **Symlinks are not followed by default** and hidden files
  are skipped — both correct defaults for a duplicate
  finder, but bite the first time you scan
  `~/Library/Application Support/` and wonder why nothing
  showed up. `--follow-links` and `--hidden` exist; read
  `fclones group --help` once before the first real run.
- **`fclones remove` on a report that was generated *days*
  ago is dangerous** — files may have been modified in
  between, in which case the cached hash no longer matches
  the on-disk content. The tool re-checks file size by
  default (cheap) but not content (expensive). Pass
  `--rf-over-prio` and re-run `group` if the corpus has
  changed since the report was written, or pass
  `--no-cache-mtime` so mtime changes invalidate the
  cached hash.
- **Performance tuning matters on HDDs.** The default
  worker pool is sized for SSDs and will thrash a single
  spinning disk. Use `--threads main=1,group=1` to
  serialize the walker and the hasher; published benchmarks
  show 3–5× speedup over the default on 7200 RPM drives.
