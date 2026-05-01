# rustic

## What it does
A fast, encrypted, deduplicated backup CLI written in Rust that is **wire-compatible
with restic repositories** — same chunking algorithm (CDC via rolling hash), same
pack format, same tree/snapshot/index objects, same AES-256 / Poly1305 encryption,
so a repo created by `restic` can be read and written by `rustic` and vice versa.
On top of that base it adds a TOML-driven `--config` profile system (so `rustic
backup` reads the source paths, exclude rules, parent-snapshot policy, retention
policy, and password-command from a `~/.config/rustic/<profile>.toml` instead of
a wall of CLI flags), a built-in `tui` browser for snapshots, a `webdav` mount
that exposes a snapshot as a read-only WebDAV share, a `mount` subcommand that
exposes one as a FUSE filesystem, and `prometheus` / `opentelemetry` exporters
for backup metrics. It supports the same backends restic does (local,
SFTP, REST, S3, B2, Azure, GCS, Swift, OpenStack, OneDrive via rclone) plus a
first-class `rclone:` driver.

## Why it's interesting
Different shape from `restic` (Go, single binary, mature, the reference
implementation — choose for ecosystem maturity and the largest community of
recipes), `borg` (Python, repo format is its own — not interchangeable, requires
exclusive lock per write — choose for single-host long-running self-managed
backups with a smaller dependency surface), `kopia` (Go, its own repo format with
content-addressable manifests, has a UI server — choose for teams that want a
managed-looking dashboard), and `duplicity` (Python, full+incremental tar
chains — choose only for legacy compatibility). `rustic` is the
*restic-compatible-with-config-files* option: pick it when you already have a
restic repo and want faster cold-cache performance, lower memory use, the
declarative profile system instead of wrapper scripts around `restic`, or when
you want to mix `restic` and `rustic` clients against the same repo from
different machines. Do **not** pick it if your team has years of `restic`
runbooks and tooling that shells out to the `restic` binary directly — the
flags differ in places.

## Niche category
Encrypted deduplicated backup CLI — restic-repo-compatible Rust reimplementation
with declarative TOML profiles.

## Repo
https://github.com/rustic-rs/rustic

## Version pinned
`v0.11.2` (latest tagged release, 2026-04-05)

## License
- SPDX: `Apache-2.0 OR MIT` (dual-licensed, user's choice)
- License files in upstream repo: `LICENSE-APACHE`, `LICENSE-MIT`

## Install
```sh
# Homebrew
brew install rustic

# Cargo (builds from source with default features: jq, prometheus,
# opentelemetry, tui, webdav)
cargo install rustic-rs --locked

# Pre-built binary via cargo-binstall (signed with minisign)
cargo binstall rustic-rs

# Arch
sudo pacman -S rustic

# Or download a release tarball from
# https://github.com/rustic-rs/rustic/releases/latest
```

## Usage examples
```sh
# Initialize a fresh local repo (prompts for the encryption password)
rustic -r /backup/repo init

# Back up a path with one-shot CLI flags
rustic -r /backup/repo backup ~/code --exclude-file ~/.rustic-excludes

# Or drive everything from a profile (lives at
# ~/.config/rustic/laptop.toml — declares repository, password-command,
# sources, excludes, retention, schedule)
rustic -P laptop backup
rustic -P laptop forget --prune          # apply the retention policy
rustic -P laptop check --read-data       # verify pack integrity end-to-end

# Browse snapshots in the built-in TUI
rustic -P laptop tui

# Mount the latest snapshot read-only via FUSE
mkdir /mnt/snap && rustic -P laptop mount latest /mnt/snap

# Or expose it over WebDAV on localhost:8000
rustic -P laptop webdav latest --address 127.0.0.1:8000

# Restore a single path from a specific snapshot
rustic -P laptop restore 8a4f2c1e:/home/me/notes ./restored-notes

# Read a repo that was originally created by `restic` — same password,
# same data, no migration step
rustic -r /existing/restic/repo snapshots

# Export Prometheus metrics from the last backup run
rustic -P laptop --metrics-prometheus-file /var/lib/node_exporter/rustic.prom backup
```
