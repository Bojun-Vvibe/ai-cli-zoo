# httm

> **Interactive ZFS / btrfs / APFS / NILFS2 / Restic snapshot file
> browser** — a Rust CLI that, given any path on a filesystem with
> snapshots, enumerates every historical version of that file
> across every snapshot it can find (local pools, remote NFS
> mounts, alternate replicas), shows them in a `Date | Size |
> Path` table, and lets you `cat` / `diff` / `restore` any
> version with one keystroke. Pinned to **v0.49.9** (released
> 2025-10-15, [LICENSE](https://github.com/kimono-koans/httm/blob/master/LICENSE),
> MPL-2.0).

Source: <https://github.com/kimono-koans/httm>

## TL;DR

`httm` ("hot tub time machine") is the missing user-facing layer
on top of ZFS / btrfs snapshots. The underlying snapshot
machinery is excellent — atomic, copy-on-write, near-zero cost —
but the day-to-day "I edited `config.toml` and want yesterday's
copy" workflow on stock ZFS is a multi-step ritual: `zfs list -t
snapshot`, pick one, `cd` into `.zfs/snapshot/<name>/`,
re-resolve the path, `cp` back. `httm` collapses that into
`httm config.toml`, which prints every snapshot version of the
file across every dataset that holds it; `httm -i` opens an
interactive picker (skim-backed) to browse, preview-diff, and
restore; `httm -d` shows a tree of changed files between live and
snapshot; `httm --restore=guard` does a cp-back that protects the
live file before overwrite.

It works on ZFS (the original target), btrfs (`snapper` /
`timeshift` snapshots), APFS (Time Machine local snapshots),
NILFS2, and Restic repositories — same UX, same flags, regardless
of which snapshot backend is actually under the path.

## Why it's interesting

Snapshot tooling has historically been split per-filesystem
(`zfs`, `btrfs`, `tmutil`, `snapper`, `restic`) and per-vendor
(TrueNAS web UI, Synology Time Backup, macOS Time Machine GUI).
None of them give you a uniform "show me every version of
**this specific path**, ranked by date, across every snapshot
source on this host" answer. `httm` is that answer: one binary,
one flag set, treats snapshots as a queryable historical layer
over the live filesystem rather than as a per-pool admin
construct.

## Install

```bash
# macOS
brew install httm

# Arch
pacman -S httm

# Cargo (any platform)
cargo install httm

# verify
httm --version    # 0.49.9
```

## Examples

```bash
# every snapshot version of one file
httm ~/.config/nvim/init.lua

# interactive browser (skim picker, preview pane, restore key)
httm -i ~/.config/nvim/init.lua

# show all files that differ between live and the most recent snapshot
httm -d ~/projects/myapp

# diff live vs the most recent snapshot version
httm --preview ~/.zshrc

# restore a snapshot version, guarding the live file (writes a backup first)
httm --restore=guard ~/.zshrc

# pipe-friendly: list snapshot mount paths for scripting
httm -n ~/important.db | head -3

# work against a remote replica's snapshots over NFS
httm --map-aliases /mnt/nas/home:/home ~/important.db
```

## Use when

- You run ZFS or btrfs at home / on a NAS and want the "I broke
  it 20 minutes ago" recovery to be one command, not a
  `.zfs/snapshot/` archeology session.
- You use Restic for backups and want per-file history queries
  without scripting `restic snapshots` + `restic dump` by hand.
- You're on macOS and want Time Machine local snapshots to be
  *queryable* (per-file, per-version) instead of only browsable
  through the Time Machine GUI.
- You're scripting an "auto-rollback on bad deploy" job and need
  a stable, parseable enumeration of historical file versions.

Skip `httm` when none of your filesystems take snapshots (it has
nothing to query — point it at ext4 and it returns empty), and
skip when your backup story is purely off-host object storage
that does not expose a snapshot API.
