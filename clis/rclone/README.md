# rclone

> **"rsync for cloud storage" — a single Go binary that syncs,
> copies, mounts, encrypts, and serves files across 70+ cloud
> object stores, SFTP/FTP/WebDAV/SMB endpoints, and local disks
> with one consistent verb set.** Pinned to **v1.73.5** (released
> 2026-04-19, [COPYING](https://github.com/rclone/rclone/blob/master/COPYING), MIT).

Source: <https://github.com/rclone/rclone>

## TL;DR

`rclone` is the answer when "copy this directory to S3 / GCS /
Azure Blob / B2 / R2 / Wasabi / Dropbox / Google Drive / OneDrive
/ a remote SFTP" turns into yet another vendor SDK and another
auth flow. One binary, one config (`rclone config`), one mental
model: every backend is a *remote* (`name:bucket/path`) and every
operation (`copy`, `sync`, `move`, `check`, `mount`, `serve`,
`bisync`) works against any of them. It does delta transfers,
checksum verification, parallel chunked uploads, server-side
copies where the backend supports it, transparent client-side
encryption (`crypt` overlay), throughput limits, partial-file
resume, and a FUSE/WinFsp `mount` that turns any remote into a
local filesystem.

## Install

```bash
# Homebrew (macOS / Linux)
brew install rclone

# Official install script (any Linux/macOS)
curl https://rclone.org/install.sh | sudo bash

# Linux package managers
# Debian / Ubuntu: apt install rclone
# Fedora: dnf install rclone
# Arch: pacman -S rclone
# Nix: nix-env -iA nixpkgs.rclone

# Windows
# scoop install rclone
# choco install rclone
# winget install Rclone.Rclone

# verify
rclone version    # rclone v1.73.5
```

## Examples

```bash
# one-time interactive setup of an S3 / GCS / Drive / Dropbox / … remote
rclone config

# mirror a local dir up to a remote, deleting extras on the destination
rclone sync ./site/ r2:my-bucket/site --progress --transfers=16

# pull a remote tree down with checksum verification and parallel transfers
rclone copy gdrive:Backups/2026-04 ./restore --check-first --progress

# mount a bucket as a local filesystem (read-write, with VFS cache)
rclone mount s3:assets ~/mnt/assets --vfs-cache-mode writes &

# expose a remote over HTTP / WebDAV / SFTP from one command
rclone serve webdav r2:public --addr :8080
```

## Use when

- You need one tool that talks to S3-compatible *and* Drive *and*
  SFTP *and* SMB without learning each vendor's CLI.
- Building a deterministic backup pipeline (pair with `restic` for
  encryption + dedup, or use `rclone crypt` overlay for filename
  + content encryption with no extra binary).
- Writing CI/release jobs that publish artifacts to multiple cloud
  providers from one script.
- Mounting a remote object store as a working directory for tools
  that only understand local paths (LLM datasets, video editors,
  scientific notebooks).

Skip `rclone` if you only ever touch one provider and their
official CLI gives you everything you need (e.g. pure `aws s3`
inside an AWS-only shop). `rclone` earns its keep the moment a
second backend appears.
