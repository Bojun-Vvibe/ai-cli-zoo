# xcp

- **Repo:** https://github.com/tarka/xcp
- **Version:** xcp-v0.24.7 (latest stable, 2026-02)
- **License:** GPL-3.0 ([LICENSE](https://github.com/tarka/xcp/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `cargo install xcp` · `brew install xcp` · `pacman -S xcp` (Arch via AUR / pikaur) · `nix run nixpkgs#xcp` · static release binaries on the GitHub release page for Linux (amd64) and macOS (amd64 + arm64)

## What it does

`xcp` is a **drop-in `cp` replacement focused on speed and a real
progress bar** — it does parallel directory traversal, batches small
files, uses platform-native fast paths (Linux `copy_file_range` with
reflinks via `--reflink=auto` on btrfs / xfs / bcachefs, macOS
`fclonefileat` on APFS), and renders a per-file + total progress bar
that classic `cp` has never had. Argument parsing is `cp`-shaped so
muscle memory transfers (`xcp -r src dst`, `xcp -a /etc /backup`),
with a few extras: `--gitignore` honours `.gitignore` rules during
recursive copy, and `--workers N` tunes the parallel I/O pool.

## Minimal usage example

```bash
# 1. Copy a file with a real progress bar (the "why xcp exists" demo)
$ xcp Ubuntu-24.04.iso /Volumes/USB/
# expected: a single-line progress bar that updates a few times per
# second showing bytes copied / total, transfer rate (e.g. "412 MB/s"),
# ETA, and the filename being written; on completion exits 0 silently

# 2. Recursive copy of a tree, in parallel, with progress
$ xcp -r ~/work/big-monorepo/ /backup/snapshot-2026-04/
# expected: progress bar reports "12,438 files / 8.2 GB copied" and the
# rate; on a 10 GbE link to a NAS or between NVMe SSDs the wall time is
# typically 2-5x faster than `cp -r` because the directory walk and the
# I/O are not serialised on one thread

# 3. Reflink-clone instead of copy on a CoW filesystem (instant, 0 bytes)
$ xcp --reflink=auto huge-vm.qcow2 huge-vm.snapshot.qcow2
# expected: completes in milliseconds even for a 100 GB file when src
# and dst are on the same btrfs / xfs / bcachefs / APFS volume; the new
# file shares blocks with the original until either is written to;
# falls back to a real copy on filesystems without reflink support

# 4. Copy a project directory while honouring .gitignore (skip node_modules / target)
$ xcp -r --gitignore ./my-project /tmp/share/
# expected: same recursive copy but build artifacts, virtualenvs, IDE
# caches, and anything else listed in .gitignore is silently skipped;
# the destination tree is what you would have committed

# 5. Tune the worker pool for a slow target (e.g. SMB share)
$ xcp -r --workers 2 ./photos/ /Volumes/NAS/Photos/
# expected: limits parallel writes to 2 to avoid hammering a network
# mount that gets slower with more concurrency; same progress bar UX
```

## When to pick it / when not to

Pick `xcp` when you actually want to **see how long a copy is going to
take** — copying a multi-GB file or a large tree with `cp` is a
silent black box, `xcp` shows a meaningful progress bar and is usually
faster on SSD-to-SSD work because it parallelises the directory walk
and uses `copy_file_range` / reflink fast paths your shell's `cp` may
not. The reflink behaviour alone is worth installing it for: `xcp
--reflink=auto big.iso big.copy.iso` on btrfs / xfs / bcachefs / APFS
is essentially free, where `cp` (without `--reflink=always`, which is
not the default everywhere) duplicates the bytes. Pair with
[`croc`](../croc/) (croc is host-to-host with NAT traversal; xcp is
local-or-mounted-volume), with [`rclone`](../rclone/) (rclone is the
right answer for cloud / S3 / B2 / SFTP transfers, xcp is for local
filesystems), with [`restic`](../restic/) (restic is dedup'd snapshots
to a backup repo, xcp is "just put a copy over there"), and with
[`fclones`](../fclones/) after a copy if you suspect the new tree has
duplicates worth reflinking.

Skip it for one tiny file — `cp` has zero startup cost and the
progress bar is overkill. Skip it for atomicity guarantees — `xcp` is
a copy tool, not a transactional file manager; if you need
"either everything copies or nothing does", wrap in a staging
directory + `mv`. Skip it for cloud / object stores — `rclone` is the
right tool there. Skip it on Windows — Linux + macOS are first-class,
Windows is not officially supported (the README explicitly warns about
reflink behaviour on Mac being limited too, so verify with `--reflink=never`
when correctness matters more than speed).
