# rmlint

> **A duplicate-file finder and lint-style cleaner that also
> identifies broken symlinks, empty files, empty directories,
> non-stripped binaries, and bad UIDs/GIDs** — written in C
> with a multi-threaded hashing pipeline (BLAKE2 / SHA / xxHash
> selectable), produces a *script* of suggested removals as its
> output rather than deleting in-place, and ships an optional
> Python+GTK GUI (`shredder`) for interactive review. Pinned
> to **v2.10.3 "Ludicrous Lemur"**
> ([COPYING](https://github.com/sahib/rmlint/blob/master/COPYING),
> GPL-3.0).

Source: <https://github.com/sahib/rmlint>

## TL;DR

`rmlint` is the answer when "I have a few terabytes of mixed
backups, photos, and downloads on this disk and I think a
third of it is duplicate" stops being a joke and starts
being a chore. Where `fdupes` (already cataloged) is a
focused duplicate-file finder, `rmlint` is the broader
"lint your filesystem" tool: same hash-based duplicate
detection (faster, multi-threaded, picks BLAKE2 by default),
plus duplicate *directories* (whole trees that are byte-for-
byte copies of another tree), empty files, empty dirs,
broken symlinks, and a handful of Unix hygiene checks. The
critical safety design is that `rmlint` never deletes
anything during the scan: the scan emits
`rmlint.sh` (a shell script of suggested `rm` / `ln` /
`reflink` operations) which you read, edit, and then run
yourself.

## Install

```bash
# Homebrew (macOS / Linux)
brew install rmlint

# Debian / Ubuntu
sudo apt install rmlint

# Arch Linux
pacman -S rmlint                  # extra repo

# Fedora
sudo dnf install rmlint

# From source (scons-based build)
git clone https://github.com/sahib/rmlint
cd rmlint
git checkout v2.10.3
scons config
scons -j4
sudo scons install

# verify
rmlint --version    # rmlint 2.10.3 (rev 0)
```

The optional `shredder` GUI is a separate Python+GTK package
(`pip install --user .` from `lib/shredder/` in the source
tree, or `pacman -S rmlint-shredder`); the CLI works without
it.

## Use it for

```bash
# Find duplicates in one tree (default config)
rmlint ~/Downloads

# Multiple trees: dupes are detected across all of them
rmlint ~/Downloads ~/Pictures /mnt/backup

# Treat the second tree as "originals to keep"; suggest
# deleting only duplicates that live in the first tree
rmlint ~/messy-copies // ~/canonical-archive

# After the scan, review and run the generated script
less rmlint.sh
sh ./rmlint.sh                    # actually delete
sh ./rmlint.sh -d                 # dry-run
sh ./rmlint.sh -p                 # paranoid byte-for-byte
                                  # re-check before each rm

# Pick a specific hash (BLAKE2b is default; xxhash for speed)
rmlint --algorithm=xxhash ~/Pictures
rmlint --algorithm=sha256 ~/Pictures      # cryptographic

# Replace duplicates with reflinks (instant, btrfs / xfs / apfs)
rmlint --config=sh:handler=reflink ~/Pictures
sh ./rmlint.sh                    # reflinks, not deletes —
                                  # disk usage drops, paths stay

# Replace duplicates with hardlinks instead of deleting
rmlint --config=sh:handler=hardlink ~/Pictures

# Find duplicate *directories* (trees whose every file matches
# another tree's every file)
rmlint --types=duplicates,duplicatedirs ~/photo-archive

# Find only empty files / empty dirs / broken symlinks
rmlint --types=emptyfiles,emptydirs,badlinks /var/lib/foo

# JSON output for piping into a script / TUI / agent
rmlint -o json:rmlint.json ~/Downloads

# Limit to files in a size range (skip the 4 GB ISOs)
rmlint --size 1M-100M ~/Pictures

# Honor xattrs to cache hashes between runs (huge speed-up
# when scanning the same tree repeatedly)
rmlint --xattr ~/photo-archive
```

The output script (`rmlint.sh`) is the safety mechanism:
the scan is read-only, the script is editable, and the
script's `-d` flag is a dry-run. For ad-hoc inspection,
`-o pretty:rmlint.txt -o json:rmlint.json` writes both a
human report and a structured one.

## Why include it in a CLI catalog

1. **It is the broader "lint my filesystem" tool, not just a
   dupe finder.** [`fdupes`](../fdupes/) and
   [`fclones`](../fclones/) (both cataloged) find duplicate
   *files*. `rmlint` finds duplicate files **plus** duplicate
   directory subtrees, empty files, empty dirs, broken
   symlinks, and bad UIDs / GIDs in one pass — closer in
   spirit to `lintian` for filesystems than to a dedupe
   tool. That single-pass property matters when each pass
   takes 40 minutes on a NAS.
2. **The "emit a script, do not delete" safety design.**
   Almost every duplicate-file finder grows an interactive
   `-d` mode that asks per-group which to keep. `rmlint`
   instead writes a shell script you can read, diff, version-
   control, and run when ready (and the script itself has a
   paranoid `-p` mode that re-hashes each file immediately
   before deleting it). For "delete a million paths"
   workflows, the audit-trail-by-default is the right
   default.
3. **First-class reflink / hardlink handlers.** With
   `--config=sh:handler=reflink`, the generated script
   replaces duplicates with reflinks (CoW shared blocks) on
   btrfs / xfs / apfs / bcachefs — disk usage drops to one
   copy, every path remains valid, and the duplicate
   relationship is filesystem-enforced (modify one path,
   the others stay intact). This is the right answer for a
   photo / media archive where renaming is expensive.

For an LLM-CLI workflow, `rmlint -o json:report.json ~/dir`
gives an agent a structured group list (each entry has
size, hash, list of paths, and an `is_original` boolean) it
can summarize ("you have 412 GB of duplicate photos in 8
groups, the largest group is 47 GB across 3 paths") without
parsing any pretty-printed output.

## Vs Already Cataloged

- **Vs [`fdupes`](../fdupes/):** closest peer — `fdupes` is
  the C classic, single-threaded, hash + byte-for-byte
  compare, output is plain stdout groups. `rmlint` is the
  multi-threaded, broader-scope, script-emitting cousin.
  For a single small tree on a laptop, `fdupes` is
  probably already installed and is enough; for a NAS-
  scale archive or any case where you want reflink dedup,
  `rmlint` is the upgrade.
- **Vs [`fclones`](../fclones/):** orthogonal-ish — `fclones`
  is the modern Rust competitor, optimized for huge file
  counts (millions of files) with parallelism and reflink
  support; `rmlint` adds duplicate-*directory* detection and
  the broader filesystem-lint surface (empty files, broken
  symlinks). Pick `fclones` for raw "dedupe a million-file
  tree fast"; pick `rmlint` when the question is wider than
  files-only.
- **Vs [`czkawka`](../czkawka/):** orthogonal — `czkawka` is
  a Rust dupe finder with a GTK GUI by default and similar
  filesystem-lint scope; `rmlint` is CLI-first with an
  optional GUI (`shredder`). Both are reasonable; teams
  already on a GTK desktop tend to land on `czkawka`,
  teams scripting from a server land on `rmlint`.
- **Vs [`dust`](../dust/) / [`dua`](../dua/):** orthogonal —
  `dust` and `dua` answer "what is using my disk space" by
  size; they do not detect duplicates. Use them first to
  find the suspicious directory, then point `rmlint` at it
  to find the duplication inside.
- **Vs [`trashy`](../trashy/):** orthogonal — `trashy` is a
  safer `rm`. After `rmlint` emits its script, you could
  edit the script to use `trash` instead of `rm` so the
  "deletes" are recoverable for a few days.

## Caveats

- **GPL-3.0 license.** Stricter than the MIT / Apache-2.0
  norm in this catalog; for redistribution inside a
  proprietary product, that matters. Pure end-user CLI use
  is unaffected.
- **The generated `rmlint.sh` is the user interface.** If
  you `sh rmlint.sh` without reading it, you are deleting
  whatever it lists. The script is well-commented, but the
  tool's safety story relies on the user opening it. The
  paranoid mode (`sh rmlint.sh -p`) re-checks byte-for-byte
  before each removal, which is the right default for one-
  off cleanups.
- **`scons` build is unusual.** Distro packages are fine;
  building from source needs `scons` (Python-based) rather
  than CMake / autotools, which trips up first-time
  builders. Use the package whenever possible.
- **Reflink handler requires a CoW filesystem.** btrfs / xfs
  (with `reflink=1`) / apfs / bcachefs / zfs (limited).
  `ext4` and most NAS filesystems do not support reflinks;
  the script falls back to hardlink or remove according to
  config.
- **Duplicate-directory detection is opt-in.** Default scan
  is files-only; `--types=duplicates,duplicatedirs` is the
  flag. Forgetting it means you only see file-level dupes
  even when whole subtrees match.
- **Last tagged release was v2.10.3 (2025-03).** Active
  bug-fix project but feature cadence is slow; the codebase
  is mature and the "Ludicrous Lemur" line has been stable
  for years.
