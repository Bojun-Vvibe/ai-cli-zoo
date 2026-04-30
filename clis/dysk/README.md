# dysk

> **A linux-first `df` replacement that knows what
> you actually meant** — a single Rust binary
> from the Canop / Broot / Bacon family that lists
> mounted filesystems with human-readable sizes,
> usage bars, mount-point annotations, and
> automatic hiding of the kernel pseudo-mounts
> (`tmpfs`, `cgroup`, `proc`, `sys`, snap loop
> devices) that pollute every `df -h` invocation.
> Pinned to **v3.6.0b**
> ([LICENSE](https://github.com/Canop/dysk/blob/main/LICENSE),
> MIT, © Denys Séguret).

Source: <https://github.com/Canop/dysk>

## TL;DR

`dysk` is what `df -h` would look like if it had
been written this decade. By default it prints
one line per *real* filesystem (block device or
network mount), each with a coloured usage bar,
human-readable used/free/total, the filesystem
type, the mount point, and a hint about the
underlying device — and it deliberately omits
the dozens of `tmpfs` / `devtmpfs` / `cgroup2` /
`squashfs` / `overlay` mounts that `df` dumps on
you and that you almost never want to see. When
you *do* want to see them, `-a` brings them all
back. Output is fully filterable with a small
expression language (`-f 'size > 100G'`,
`-f 'use > 75%'`, `-f 'fs=ext4'`, `-f 'remote'`)
and fully scriptable as JSON or CSV
(`--json`, `--csv`) so you can feed it into a
monitoring pipeline without re-parsing `df`'s
column-aligned text. There's also a "stats" mode
(`--stats`) that summarises across all mounts
(total used / total free / inode usage), and a
"cols" flag that lets you redesign the table
(`-c +disk-mount-use-free,-fs` to add a disk
column and drop the fs column). Everything is
read out of `/proc/mounts` and the `statvfs(2)`
syscall, so it's instant — no shelling out, no
network calls, no init scripts.

## Install

```bash
# Cargo (any platform with Rust toolchain — Linux is the
# primary target; macOS works for read-only listings)
cargo install dysk --locked

# pre-built static binaries on the releases page
# https://github.com/Canop/dysk/releases
curl -L https://github.com/Canop/dysk/releases/latest/download/dysk_linux.tar.gz \
  | tar -xz -C /usr/local/bin

# Arch Linux (AUR)
yay -S dysk

# Nix
nix-env -iA nixpkgs.dysk

# Debian / Ubuntu (community .deb on releases page)
curl -L -o dysk.deb https://github.com/Canop/dysk/releases/latest/download/dysk.deb
sudo dpkg -i dysk.deb

# verify
dysk --version    # dysk 3.6.0
```

## Basic usage

```bash
# default — only "real" filesystems, one row each, with usage bars
dysk

# include kernel pseudo-mounts (tmpfs, cgroup, proc, sys, etc.)
dysk -a

# only the filesystems above 75 % full — perfect for "what's about to fill?"
dysk -f 'use > 75%'

# only filesystems larger than 100 GB
dysk -f 'size > 100G'

# only ext4 mounts
dysk -f 'fs = ext4'

# only network filesystems (nfs, cifs, sshfs, fuse.s3fs ...)
dysk -f 'remote'

# combined: large local filesystems above 50 % full
dysk -f 'size > 50G & use > 50% & !remote'

# JSON for scripts / dashboards
dysk --json | jq '.[] | select(.use_percent > 80)'

# CSV for spreadsheets
dysk --csv > storage-snapshot.csv

# inode usage instead of byte usage
dysk -i

# add / remove columns
dysk -c +disk-mount-use-free,-fs

# whole-host roll-up
dysk --stats

# follow a specific mount
dysk /var
```

## When to choose

- **You SSH into a box and the first thing you do
  is `df -h | grep -v tmpfs`** — `dysk` is that
  pipeline made into the default. You see real
  filesystems, you see usage bars, and you see
  the mount-point hierarchy at a glance, without
  the wall of `tmpfs` lines.
- **You're writing a disk-fullness alert and you
  don't want to parse `df`'s columns** —
  `dysk -f 'use > 90%' --json` is the canonical
  shell-friendly answer; the schema is stable
  across releases and the filter expression
  language means you push the predicate down
  rather than `awk`-ing the output.
- **You operate a fleet with a mix of LVM, btrfs
  subvolumes, ZFS, NFS, and bind-mounts** — the
  `disk` column resolves logical mounts back to
  underlying block devices so you can spot two
  mount points sharing one disk before they
  both run out of space.
- **You want a single binary that drops cleanly
  into a minimal container image** — static Rust
  build, no Python, no shell, no package-manager
  metadata.

## When NOT to choose

- **You're on macOS or BSD as a primary target** —
  `dysk` reads `/proc/mounts`, which is Linux-only.
  It builds on macOS and will list mounts via
  `getmntinfo(3)`, but the feature set is reduced
  (no inode column, no LVM-aware disk mapping).
  On macOS the zoo's [`duf`](../duf/) is a better
  fit. On BSD, native `df` already does most of
  the right things.
- **You want directory-level "what's eating my
  disk?"** — that's a `du` problem, not a `df`
  problem. Use [`dust`](../dust/),
  [`dua`](../dua/), or
  [`broot`](../broot/) `:du`.
- **You need POSIX `df` output for a legacy
  script** — `dysk` does not emulate `df`'s exact
  columns. Either keep `df` for that script or
  use `dysk --csv` and translate.
- **You only have one disk and one mount** —
  `df -h` is fine. The win from `dysk` scales
  with the number of mounts you have to ignore.

## Why it fits the zoo

The zoo's storage-introspection cluster
([`dust`](../dust/),
[`dua`](../dua/),
[`duf`](../duf/),
[`erdtree`](../erdtree/) — actually
[`broot`](../broot/) `:du` for the tree-view
case) covers "where did my space go?" at the
file/directory level. `dysk` covers the
mount-point level — the layer above. It's
also another piece of the Canop ecosystem
already represented by [`broot`](../broot/) and
[`bacon`](../bacon/) — the same author's
recognisable house style of "boring CLI flags,
expressive filter syntax, structured output for
scripts, sensible defaults for humans."

## Upstream pointers

- Repo: <https://github.com/Canop/dysk>
- Release notes: <https://github.com/Canop/dysk/releases>
- License: [MIT](https://github.com/Canop/dysk/blob/main/LICENSE)
- Documentation: <https://dystroy.org/dysk/>
- Filter expression reference:
  <https://dystroy.org/dysk/filter/>
- Author: [@Canop](https://github.com/Canop)
  (Denys Séguret), also [`broot`](../broot/) and
  [`bacon`](../bacon/).
- Previous name: `lfs`. Renamed to `dysk` in 2.x.
