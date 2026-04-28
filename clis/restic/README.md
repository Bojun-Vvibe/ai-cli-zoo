# restic

> **Fast, secure, efficient backup program** — a single Go binary
> that produces deduplicated, encrypted, content-addressed
> snapshots of your data into a *repository* hosted on local disk,
> SFTP, S3-compatible object storage, Azure Blob, GCS, B2,
> Swift, REST server, or anything `rclone` can reach. Pinned to
> **v0.18.1** (released 2025-09-21,
> [LICENSE](https://github.com/restic/restic/blob/master/LICENSE),
> BSD-2-Clause).

Source: <https://github.com/restic/restic>

## TL;DR

`restic` is the answer when you want backups that are
*verifiable*, *encrypted by default*, and *cheap to keep* without
running a full backup server. It chunks files with a rolling hash
(content-defined chunking, ~1 MiB average), encrypts each chunk
with AES-256 + Poly1305 under a key derived from your repo
password, and stores them once — so a 100 GiB tree backed up
nightly costs roughly the size of the daily *delta*, not 100 GiB
× N. Snapshots are immutable, restorable to any point, and
checked end-to-end with `restic check --read-data`. The same
binary on macOS / Linux / Windows / *BSD writes to the same
repository format, so you can back up a laptop and restore on a
server.

## Install

```bash
# Homebrew (macOS / Linux)
brew install restic

# Linux package managers
# Debian / Ubuntu: apt install restic
# Fedora: dnf install restic
# Arch: pacman -S restic
# Nix: nix-env -iA nixpkgs.restic

# Windows
# scoop install restic
# choco install restic
# winget install restic.restic

# Static binary (any OS, no deps)
# https://github.com/restic/restic/releases

# verify
restic version    # restic 0.18.1 compiled with go1.x.x on …
```

## Examples

```bash
# initialise an encrypted repo on local disk (or s3:..., sftp:..., b2:...)
export RESTIC_REPOSITORY=/mnt/backups/laptop
export RESTIC_PASSWORD_FILE=~/.restic-pass
restic init

# back up a tree, excluding noisy paths
restic backup ~/Documents ~/Projects \
  --exclude-caches \
  --exclude '*/node_modules' \
  --exclude '*/.venv'

# list snapshots, then restore one to a scratch dir
restic snapshots
restic restore latest --target /tmp/restore --include /Documents

# prune old snapshots with a retention policy and reclaim space
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune

# verify repo integrity (sample data blocks, not just metadata)
restic check --read-data-subset=5%
```

## Use when

- You want one tool that handles dedup + encryption + remote
  upload, and you do not want to run a backup *server* (no
  Bacula / Bareos / BorgBackup-server-side daemon).
- Backups must be readable on a different OS than the one that
  wrote them (laptop on macOS, restore on Linux box).
- The destination is object storage (S3, B2, R2, Wasabi, GCS,
  Azure Blob) and you care about cost — content-addressed dedup
  + immutable snapshots map well to lifecycle rules and
  Object Lock for ransomware protection.
- Pair with `rclone serve restic` to back up to *any* of
  rclone's 70+ backends, or use the native `rest-server` for a
  hardened append-only endpoint.

Skip `restic` for "I just need a tarball nightly" — `tar` + a
cron + `rclone copy` is fine. `restic` earns its keep when
retention, dedup, encryption, and verifiability all matter at
once.
