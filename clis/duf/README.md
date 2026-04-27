# duf

> **Disk Usage / Free Utility — a friendlier `df`** — a single Go
> binary that renders mounted filesystems as a colour-coded,
> sortable, grouped table with usage bars, mount points, inode
> stats, and per-fs-type filtering. Pinned to **v0.9.1** (commit
> `b2f01a1ecbfd1af63015b83296c50a95260f97b1`,
> [LICENSE](https://github.com/muesli/duf/blob/master/LICENSE),
> MIT).

Source: <https://github.com/muesli/duf>

## TL;DR

`duf` is what you reach for when `df -h` produces a wall of
loopback / overlay / squashfs / tmpfs lines and you have to
squint to find the one real disk that's at 96 %. It groups
filesystems by category (local / network / fuse / special),
shows usage bars instead of just percentages, sorts by any
column, and filters with `--only`/`--hide` flags. Same
underlying `statfs` data as `df`, ~10× faster to read.

## Install

```bash
# Homebrew (macOS / Linux)
brew install duf

# Linux package managers
# Arch:    pacman -S duf
# Debian:  apt install duf      (bookworm+)
# Fedora:  dnf install duf
# Nix:     nix-env -iA nixpkgs.duf
# Alpine:  apk add duf

# Go install (any OS with a Go toolchain)
go install github.com/muesli/duf@latest

# from a release archive (any OS)
curl -Lo duf.tar.gz "https://github.com/muesli/duf/releases/download/v0.9.1/duf_0.9.1_Darwin_arm64.tar.gz"
tar xf duf.tar.gz
sudo install duf /usr/local/bin/

# verify
duf --version    # duf 0.9.1
```

No config file, no shell hook — single binary.

## License

MIT — see
[LICENSE](https://github.com/muesli/duf/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. default view: grouped tables (local / network / special)
duf

# 2. only local disks (the most common follow-up)
duf --only local

# 3. hide the noisy bind-mount / overlay / squash entries
duf --hide special,loops,binds

# 4. sort by usage descending — find the disk that's about to fill
duf --sort usage --output mountpoint,size,used,avail,usage

# 5. inode pressure (separate problem from byte pressure)
duf --inodes

# 6. JSON output for scripting / dashboards
duf --json | jq '.[] | select(.usage > 0.9) | .mount_point'

# 7. narrow to a specific filesystem type
duf --only-fs ext4,xfs,zfs

# 8. one-mount drill-down (handy in CI when "/tmp" is the suspect)
duf /tmp /var/lib/docker

# 9. theme it for your terminal
duf --theme dark        # default; also: light, ansi
duf --style unicode     # also: ascii (for restrictive terminals)
```

## Niche It Fills

**`df`'s output is honest but unreadable in 2026.** A modern
laptop has ~30 mounts (overlayfs from Docker, squashfs from
Snap, tmpfs for `/run/*`, fuse mounts from `gocryptfs` /
`sshfs`, autofs, the actual root); `df -h` lists them all
in alphabetical mount-point order with no grouping. `duf`
keeps the same data but groups, colours, sorts, and filters
it — so the question "which of my real disks is full" goes
from "scan 30 lines" to "look at the one row in the local
table whose bar is red".

## Why use it

Three things `duf` does that `df` does not:

1. **Grouped tables, not one giant list.** Local disks,
   network mounts (NFS / SMB / SSHFS), fuse mounts, and
   special filesystems (tmpfs, devtmpfs, cgroup, proc,
   overlay) each get their own table. The cognitive load of
   "is this row real or is it a bind-mount" goes away.
2. **Colour-coded usage bars.** A 96 %-full disk is a red bar;
   a 40 %-full one is green. The percentage column in `df`
   requires you to compare numbers; `duf` lets you spot the
   problem with peripheral vision.
3. **`--json` and `--only`/`--hide` filters make it scriptable.**
   `duf --only local --json | jq '.[] | select(.usage > 0.85)'`
   is a one-liner monitoring check; doing the equivalent with
   `df --output=...` plus `awk` requires careful column
   discipline. The JSON shape is stable across releases.

For agent / LLM workflows where the model is doing on-host
sysadmin (free up space before a build, decide whether to
prune Docker images), `duf --json` returns a parseable
view of disk pressure that the agent can act on without
parsing `df`'s human-formatted columns.

## Vs Already Cataloged

- **Vs [`dust`](../dust/):** complementary — `dust` answers
  "*within* this directory tree, where are the bytes" (recursive
  size walk per subdirectory); `duf` answers "*across* my
  mounted filesystems, which one is full". The pair: `duf` to
  find the saturated mount, `dust` to find the directory inside
  it that's the culprit.
- **Vs [`procs`](../procs/):** orthogonal — `procs` is a `ps`
  replacement (per-process view); `duf` is a `df` replacement
  (per-filesystem view). Both share the "modern Rust/Go
  rewrite of a coreutils tool with colour + grouping" theme.
- **Vs [`bottom`](../bottom/):** `bottom` (`btm`) is a full-
  screen TUI dashboard for CPU / mem / net / disk / processes,
  intended to stay open like `htop`. `duf` is a one-shot
  printout you run, read, and exit from. Use `bottom` for
  monitoring over time, `duf` for "is this disk full right now,
  yes or no".
- **Vs `df` / `df -h` / `df -hT` directly:** `df` wins inside
  POSIX scripts where you need exit codes and the same output
  on every platform back to the 1980s. `duf` wins for
  interactive use and for modern JSON-consuming scripts.

## Caveats

- **Mount detection is platform-specific.** Linux / macOS /
  BSD / Windows each use a different syscall path; `duf`
  abstracts them well in the common case but exotic setups
  (NetBSD jails, illumos zones, Windows mounted VHDs) can
  show up in the wrong group or with missing columns. Cross-
  check against `df` if a number looks suspicious.
- **`--only`/`--hide` accept categories, not regex.** You can
  say `--hide special` or `--hide-fs squashfs,overlay` but you
  cannot say `--hide-mount '^/snap/'` directly; use a shell
  filter on `--json` for that case.
- **Inode and byte views are separate flags.** `duf` defaults
  to bytes; `--inodes` switches to inode usage. There's no
  single combined view, so a filesystem that's at 99 % inodes
  but 20 % bytes (lots of tiny files) won't ring an alarm in
  default mode. Run both checks in monitoring.
- **No "watch" / refresh mode.** It's a one-shot; for
  continuous monitoring use `watch -n 5 duf`, or pick
  [`bottom`](../bottom/) instead. (`watch` strips colour
  unless you pass `watch -c`.)
- **Release cadence is slow.** v0.9.1 is from late 2025; the
  tool is feature-complete enough that this isn't a problem,
  but new filesystem types (e.g. bcachefs categories) may
  show up as "unknown" until a future release re-classifies
  them.
