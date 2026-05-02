# duplicacy

> **Lock-free deduplicating cloud backup tool with a "two-step
> variable-size chunking" algorithm that lets multiple clients
> back up to the same storage without any coordinating server**
> — written in Go, single static binary, supports a dozen storage
> backends out of the box. Pinned to **v3.2.4**
> ([LICENSE](https://github.com/gilbertchen/duplicacy/blob/master/LICENSE),
> custom source-available license — free for personal/CLI use,
> commercial use of the GUI requires a paid license).

Source: <https://github.com/gilbertchen/duplicacy>

## TL;DR

`duplicacy` is the answer to "I want `restic`-style deduplicated
encrypted backups, but I want multiple machines pushing into the
*same* repository without a central locking server, and I want a
storage-format that survives the loss of any one snapshot." Its
core trick is **lock-free deduplication**: chunk IDs are derived
deterministically from chunk content (HMAC-SHA-256 of the data
under a per-repository key), each chunk is uploaded under its own
hash-named path, and snapshot files reference chunk hashes — so
two clients backing up the same file at the same time both upload
the same content-addressed chunk and the second upload is a no-op,
without any S3 lock, without any DynamoDB-style coordinator. The
storage backends include local disk, SFTP, S3 (and any
S3-compatible: Backblaze B2 native + S3, Wasabi, MinIO, Storj,
Cloudflare R2), Google Drive, OneDrive, Dropbox,
WebDAV, Hubic, OpenStack Swift, and Google Cloud Storage —
configured via `duplicacy add` and stored as plain JSON in
`.duplicacy/preferences`. Erasure coding (configurable
Reed-Solomon shards) optionally protects each chunk against
silent storage corruption, RSA-OAEP encryption with a per-repo
master key handles confidentiality, and `duplicacy prune` does
content-addressed garbage collection that is itself lock-free
(uses a fossil-collection algorithm: chunks marked unreferenced
become "fossils" that survive one more backup cycle before
deletion, so a concurrent `backup` from another client cannot
race the GC).

## Install

```bash
# Homebrew (macOS / Linux)
brew install duplicacy

# Direct binary (Linux / macOS / FreeBSD / Windows)
curl -L https://github.com/gilbertchen/duplicacy/releases/download/v3.2.4/duplicacy_osx_arm64_3.2.4 \
    -o /usr/local/bin/duplicacy && chmod +x /usr/local/bin/duplicacy

# Go (build from source — needs Go 1.21+)
git clone https://github.com/gilbertchen/duplicacy
cd duplicacy && go build -o duplicacy duplicacy/duplicacy_main.go

# verify
duplicacy --version
```

## Example usage

```bash
# init a new repository, point it at a B2 bucket
cd ~/Documents
duplicacy init mylaptop b2://my-backup-bucket
# (prompts for B2 keyId + applicationKey, then for an
# encryption password — both stored in ~/.duplicacy/preferences
# encrypted with the OS keychain when -e is passed)

# first backup — uploads everything
duplicacy backup -stats

# subsequent backups — only changed chunks travel
duplicacy backup -stats

# add a second storage backend for 3-2-1 redundancy
duplicacy add offsite mylaptop sftp://user@nas.local/backups

# copy snapshots from primary to secondary (chunk-level,
# resumable, deduplicating against any chunks already at dest)
duplicacy copy -from default -to offsite

# list snapshots
duplicacy list

# restore a specific revision into a new directory
duplicacy restore -r 42 -- 'Documents/important.pdf'

# prune snapshots older than 30 days, keep weeklies for a year
duplicacy prune -keep 0:365 -keep 7:30 -keep 1:7
```

## When to choose vs alternatives

Pick **duplicacy** over [`restic`](../restic/) when multiple
machines need to push into the *same* repository concurrently —
restic uses a repository lock that serialises writes (one client
backing up blocks another), duplicacy's lock-free design means N
laptops + a server + a NAS can all back up to the same B2 bucket
in parallel and dedupe against each other's chunks. Pick over
restic when erasure coding matters (restic does not have built-in
Reed-Solomon; duplicacy does). Pick **restic** instead when the
repository is single-writer, when the GPL-compatible BSD-2 license
is a hard requirement (duplicacy's license is custom and the GUI
is paid), or when the broader plugin/tooling ecosystem matters
(`autorestic`, `resticprofile`, `backrest` UI). Pick over
[`borgbackup`](../borgbackup/) when the storage target is cloud
object storage (borg requires SSH-accessible storage or the
`borg serve` daemon — no native S3); pick borg when the workflow
is "one client, one server, SSH" and you want the most mature
single-writer dedup story. Pick over [`rclone`](../rclone/) `+`
`rsync` for **versioned** backups (rclone is a sync tool, not a
snapshot tool — it cannot give you "the file as it was 30 days
ago"). Pick over [`kopia`](https://kopia.io) when CLI-first +
lock-free multi-client + erasure coding wins; kopia wins when you
want a polished GUI + scheduling daemon out of the box. Skip
when single-machine + single-target + no encryption is enough
(plain `rsync --link-dest` snapshots are simpler).

## Caveats

- **License**: source-available, free for CLI use; commercial use
  of the duplicacy GUI / Web edition is paid. Check the
  [LICENSE](https://github.com/gilbertchen/duplicacy/blob/master/LICENSE)
  before redistributing or building it into a paid product.
- **Fossil collection requires periodic backups from every client**
  to safely garbage-collect — if a laptop goes offline for months,
  pruning may be deferred until it checks in (or the operator
  uses `-exclusive` to force a serialised prune).
- **Chunk size tuning matters for cloud bills**: the default
  variable-size chunker averages ~4 MB chunks; very large repos
  benefit from `-chunk-size 16M` to reduce per-object request
  overhead on B2 / S3.
