# borg (BorgBackup)

- **Repo:** https://github.com/borgbackup/borg
- **Version:** 1.4.4 (tagged 2026-03-19; current stable on the 1.4 line — a refreshed 1.2 with a few breaking-but-tested changes; the 2.x line is in beta and is a separate repo posture)
- **License:** BSD-3-Clause — see [`LICENSE`](https://github.com/borgbackup/borg/blob/master/LICENSE)
- **Language:** Python (with C extensions for the chunker, hash, and compression hot paths) — but shipped as a self-contained PyInstaller "fat binary" so the runtime end-user installs *one* file
- **Install:** `brew install borgbackup` · `apt install borgbackup` · `pacman -S borg` · `dnf install borgbackup` · `pip install "borgbackup==1.4.4"` · or download the signed `borg-linux-glibc235-x86_64-gh` / `borg-macos-15-arm64-gh` / `borg-freebsd-14-x86_64-gh` fat-binary from the GitHub release page (PGP-signed; `.asc` files alongside) — binary name is `borg`

## Overview

`borg` is a single-binary deduplicating backup tool that
chunks files with a content-defined Buzhash rolling
hasher, hashes each chunk with HMAC-SHA-256 (or BLAKE2b
under `--encryption=authenticated-blake2`), and stores
each unique chunk **once** in a content-addressable repo
on the local filesystem or over an SSH-tunnelled `borg
serve` on a remote host. The result: a daily snapshot of
a 200 GB working tree typically writes a few hundred MB
of new chunks even when most files have changed in place,
because the chunker's boundaries are stable across small
edits. Backups are encrypted client-side by default
(`repokey-blake2` keeps the key in the repo wrapped by
your passphrase; `keyfile-blake2` keeps the key only on
the client), so the storage host — including untrusted
SFTP hosts and friend-of-friend "borgbase" providers —
never sees plaintext. `borg check` verifies repo
integrity end-to-end; `borg mount` exposes any archive
as a FUSE filesystem you can `cd` into and `cp` out of;
`borg prune` enforces retention policies (`--keep-daily 7
--keep-weekly 4 --keep-monthly 12 --keep-yearly 5`)
without ever needing to read or rewrite the underlying
chunks.

## Niche

**Encrypted, deduplicating, append-only client-side
backup over SSH**, with retention pruning, FUSE-mounted
restore, and a single self-contained binary on every
end-user OS. The role is "the engine you point `cron` at
to back up `/home`, `/etc`, `/var/lib/postgres`, and an
LLM project tree to a cheap remote box". The competing
universe is `restic` / `kopia` / `duplicity` / `bup` /
`tarsnap` — see comparisons below.

## When to use

- You want client-side encryption + dedup + compression
  in one tool, with no cloud account and no SaaS control
  plane: `ssh user@backup-host borg init --encryption
  repokey-blake2 ./repo` and `borg create ssh://user@
  backup-host/./repo::{hostname}-{now} ~` is the entire
  setup.
- The backup target is an SSH-reachable box (any VPS,
  any NAS that accepts SSH, BorgBase, rsync.net's borg
  endpoint) and you want push-from-client semantics.
- Daily/hourly snapshots of a working tree where most
  files change in place — borg's content-defined
  chunking means the new-chunk volume scales with
  *actual* edits, not file mtimes.
- You need FUSE-mounted random-access restore: `borg
  mount repo::archive /mnt/borg-restore` and `cp` what
  you want without unpacking the whole archive.
- You need a backup format you can verify offline:
  `borg check --verify-data repo` reads every chunk and
  re-checks the HMAC.

## When NOT to use

- The backup target is **object storage** (S3 / R2 /
  B2 / GCS / Azure Blob) — borg's repo format wants a
  filesystem with `rename()` semantics. Use [`restic`]
  or [`kopia`] which speak object-storage natively
  (or wrap borg with `rclone mount`, but that is a
  workaround, not the design).
- You need **multi-client concurrent writes to the same
  repo** — borg locks the repo per-write. Restic and
  Kopia both support concurrent clients with an
  online lock service; borg expects one writer at a
  time.
- You are on the borg 2.x beta and want forward-
  compat: 2.x breaks repo format. Stay on 1.4.x for
  production until 2.x ships stable.
- You want a backup tool with a built-in **GUI / web
  dashboard** out of the box — borg is CLI-first;
  Vorta (third-party Qt frontend) and Pika exist but
  are separate projects. Kopia ships a web UI in-box.
- You want **forever-incremental forever-restorable**
  with a vendor-managed key escrow story — pick a
  managed service (Tarsnap, BorgBase managed, AWS
  Backup) instead of self-hosting.

## Comparison vs alternatives in zoo

- [`restic`](../restic/) — Go single binary, similar
  dedup model (CDC chunker + content-addressable repo),
  but speaks S3 / B2 / GCS / Azure / SFTP / REST natively
  and supports concurrent clients with `unlock` semantics.
  Pick `restic` when the target is object storage or
  multi-client; pick `borg` when the target is an SSH
  host and you want the more mature FUSE mount and the
  smaller per-snapshot overhead on huge file counts.
- [`kopia`](../kopia/) — Go, multi-client by default,
  built-in web UI, supports object storage and SFTP,
  policy-based retention. Pick `kopia` when you want a
  GUI and policy DSL out of the box; pick `borg` when
  CLI-only and one-writer is the right shape.
- [`rclone`](../rclone/) — sync, not backup: mirrors a
  source tree to a destination, no dedup, no snapshots,
  no point-in-time restore. Complementary — `rclone
  serve sftp` can be the borg target, or `rclone copy`
  can replicate a borg repo offsite.
- [`age`](../age/) — file-level encryption only. Pair
  with `tar` for simple encrypted archives; reach for
  `borg` when you need *snapshots* and *dedup* on top.
- [`talisman`](../talisman/) — pre-commit secret
  scanner; orthogonal (prevents secrets from entering
  the source tree, doesn't back the tree up).

## Why it earns a slot in an AI-native workflow

LLM-driven projects accumulate state that is expensive
to recompute and expensive to lose: vector indexes
(GBs of HNSW segments), fine-tune checkpoint shards,
RAG corpora, conversation logs, agent traces, the
`models/` directory full of GGUF / safetensors weights,
and `~/.config/<agent>/` settings. These directories
edit-in-place constantly (HNSW segment merges, AOF
rewrites, checkpoint resumes), which is exactly the
shape `borg`'s content-defined chunker handles best —
a daily snapshot of a 50 GB project tree typically
writes <500 MB of new chunks. The encryption-at-rest
default also matters: agent traces frequently contain
verbatim prompts and tool-call arguments that you do
not want sitting in plaintext on a third-party SFTP
host. `borg create ... --exclude-caches --exclude
'**/.venv' --exclude '**/node_modules' --exclude
'**/__pycache__'` plus a `borg prune --keep-daily 7
--keep-weekly 4` cron is a five-line ops layer that
covers the whole story.

## Example invocations

```bash
# Initialise an encrypted repo on a remote SSH host
borg init --encryption=repokey-blake2 \
     ssh://backup@host/./repo

# Create a snapshot, named after host + ISO timestamp
borg create --stats --progress --compression zstd,6 \
     --exclude-caches \
     --exclude '**/.venv' --exclude '**/node_modules' \
     ssh://backup@host/./repo::'{hostname}-{now}' \
     ~/projects ~/.config /etc

# List archives, list contents of one archive
borg list ssh://backup@host/./repo
borg list ssh://backup@host/./repo::myhost-2026-04-30T03:00:00

# FUSE-mount an archive for random-access restore
mkdir /mnt/restore
borg mount ssh://backup@host/./repo::myhost-2026-04-30T03:00:00 \
     /mnt/restore
# ... cp -a /mnt/restore/home/me/projects/foo ./
borg umount /mnt/restore

# Restore one path without mounting
borg extract ssh://backup@host/./repo::myhost-2026-04-30T03:00:00 \
     home/me/projects/foo

# Enforce retention
borg prune --list --keep-daily 7 --keep-weekly 4 \
     --keep-monthly 12 --keep-yearly 5 \
     ssh://backup@host/./repo

# Reclaim freed chunks (1.4 still uses prune+compact)
borg compact ssh://backup@host/./repo

# Verify repo integrity end-to-end
borg check --verify-data ssh://backup@host/./repo
```
