# kopia

> **Encrypted, deduplicated, compressed snapshot backups to any
> object store you point at** — a Go-implemented backup engine
> that splits files into content-addressed variable-size chunks
> (rolling-hash splitter, ~4 MB target), encrypts every chunk with
> AES-256-GCM / ChaCha20-Poly1305 under a key derived from your
> repository password, deduplicates across snapshots and across
> *machines* sharing the repo, and stores the resulting blobs in
> any of S3 / B2 / GCS / Azure / SFTP / WebDAV / rclone-remote /
> local FS / Filesystem over a network share. One static binary
> is the CLI, the maintenance worker, and the optional WebUI
> (`kopia server start`). Pinned to **v0.22.3** (released 2025-12-02,
> [LICENSE](https://github.com/kopia/kopia/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/kopia/kopia>

## TL;DR

`kopia` is what you reach for when `restic` is the obvious sibling
but you also want a real WebUI, scheduled snapshots without `cron`,
and per-source retention policies declared in YAML rather than
shell-loops over `forget`. Run `kopia repository create
s3 --bucket=mybackup --access-key=$AWS_ACCESS_KEY_ID
--secret-access-key=$AWS_SECRET_ACCESS_KEY` once, then `kopia
snapshot create ~/Documents` from any machine that knows the
repository password and dedup happens automatically — re-snapshotting
the same 200 GB tree the next day uploads only the changed chunks.
`kopia policy set --global --keep-latest=10 --keep-hourly=24
--keep-daily=14 --keep-weekly=8 --keep-monthly=12` declares the
retention; the maintenance worker prunes on a schedule. Restores
are `kopia snapshot restore <snapshot-id> /tmp/restored/` or, for
ad-hoc browsing, `kopia mount <snapshot-id> /mnt/snap` to FUSE-mount
the snapshot read-only.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kopia

# Direct binary download (Linux x64 example)
curl -LO https://github.com/kopia/kopia/releases/download/v0.22.3/kopia-0.22.3-linux-x64.tar.gz
tar xzf kopia-0.22.3-linux-x64.tar.gz && sudo install kopia /usr/local/bin/

# Debian / Ubuntu (official repo)
curl -s https://kopia.io/signing-key | sudo gpg --dearmor -o /etc/apt/keyrings/kopia-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kopia-keyring.gpg] https://packages.kopia.io/apt/ stable main" | sudo tee /etc/apt/sources.list.d/kopia.list
sudo apt update && sudo apt install kopia

# Windows
winget install kopia.kopia
```

## Usage

```bash
# 1. Create or connect to an S3-backed repo (one-time per machine)
kopia repository create s3 \
    --bucket=mybackup-prod \
    --prefix=hostA/ \
    --access-key=$AWS_ACCESS_KEY_ID \
    --secret-access-key=$AWS_SECRET_ACCESS_KEY

# 2. Snapshot a directory
kopia snapshot create ~/Projects ~/Documents

# 3. List snapshots
kopia snapshot list

# 4. Set a retention policy for one source
kopia policy set ~/Projects \
    --keep-latest=20 --keep-daily=14 --keep-weekly=8 --keep-monthly=12

# 5. Mount a snapshot read-only for browsing
kopia mount k1234abcd... /mnt/snap

# 6. Start the WebUI on localhost:51515 (requires htpasswd auth)
kopia server start --address=127.0.0.1:51515 \
    --tls-generate-cert --server-username=admin --server-password=$PASS
```

## Why pick this

`restic` and `borg` cover the same dedup + encrypt + snapshot
shape; `kopia` differentiates on (1) **first-class WebUI** that
ships in the same binary so non-CLI teammates can browse / restore
without a separate product, (2) **per-source policies in YAML**
(`kopia policy show <path>`) instead of shell-loops over `forget
--keep-*`, (3) **content-defined chunking with a rolling hash**
that dedups effectively across renames + moves + small edits where
fixed-block schemes lose the dedup, (4) **broad object-store
support** including any rclone backend so you can target Backblaze
B2, R2, Wasabi, MinIO, even Google Drive without a separate proxy.
For a homelab or small-team workload that wants `restic`-class
dedup + Apache-2.0 license + a real UI, this is the install-once-
and-forget option.
