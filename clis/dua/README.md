# dua

> **A Rust disk-usage analyzer that scans in parallel by default and
> ships an interactive TUI (`dua i`) for "navigate the tree, mark
> entries, delete with confirmation" — the keyboard-driven middle
> ground between `du -sh *` and a full file manager.** Pinned to
> **v2.34.0** (homebrew `dua-cli`),
> [LICENSE](https://github.com/Byron/dua-cli/blob/main/LICENSE),
> MIT.

Source: <https://github.com/Byron/dua-cli>

## TL;DR

`dua` ("disk usage analyzer") is what you reach for when `du -sh *`
is too slow and `ncdu` is too modal. Two surfaces in one binary:
`dua <path>` prints sorted aggregate sizes (one parallel scan, no
TUI) and `dua i <path>` drops into an interactive Miller-columns
view where `j`/`k` move, `o` enters a directory, `u` ascends,
`s` re-sorts, `m` marks entries, and `Ctrl-r` deletes everything
marked after a confirmation prompt. The scanner is `jwalk` +
`rayon` so a cold scan of `~` on an SSD finishes in single-digit
seconds where `du -sh ~/*` blocks for tens of seconds. Output
modes include byte-accurate (`-f bytes`), block-accurate
(`-f blocks` for "what does the filesystem actually charge"),
and apparent vs allocated (`-A`). The aggregate command also
takes multiple paths (`dua a b c`) and emits a sorted ranking,
which makes it a one-shot answer to "which of these N
directories is fattest" without piping into `sort -h`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install dua-cli       # currently 2.34.0

# Cargo (any platform with a Rust toolchain)
cargo install dua-cli

# Pre-built binaries for every platform
# https://github.com/Byron/dua-cli/releases

# Verify
dua --version              # dua 2.34.0
```

## License

MIT — see
[LICENSE](https://github.com/Byron/dua-cli/blob/main/LICENSE).
Permissive: vendor, fork, ship inside a commercial product
without copyleft obligations.

## One Concrete Example

```bash
# 1. quick "what is eating my home dir" — sorted aggregate, no TUI
dua ~/Downloads ~/Projects ~/Library/Caches
# prints three lines, largest first, in human-readable units

# 2. interactive cleanup pass — TUI, mark with `m`, delete with Ctrl-r
dua i ~

# 3. find the top 20 fattest leaf directories anywhere under the repo,
#    block-accurate so the numbers match `df`
dua --stay-on-filesystem -f blocks ~/Projects/foo \
  | head -20

# 4. apparent vs allocated for sparse files (VM images, DB files)
dua -A ~/VirtualBox\ VMs       # apparent size
dua    ~/VirtualBox\ VMs       # allocated size
# the gap is the sparse hole — useful before tarballing

# 5. CI guard — fail if the build cache exceeds 5 GB
test "$(dua --format bytes target | awk '{print $1}')" -lt 5000000000
```

## Niche It Fills

**The parallel-scan, TUI-optional disk-usage analyzer.** Same
family as `du`, [`ncdu`](../ncdu/), [`gdu`](../gdu/),
[`dust`](../dust/), [`dust`](../dust/), [`duf`](../duf/) —
`dua`'s specific corner is *one binary, two modes (one-shot
aggregate **and** interactive TUI), parallel by default, marked-
batch deletion from inside the TUI*. Pick when you want
`ncdu`-style cleanup ergonomics without the modal "scan, then
browse" two-step, or when you want a scriptable aggregate
command and an interactive cleanup tool that share a binary.

## Why use it

Three things `dua` does that pay off immediately:

1. **Parallel scan, single binary.** `jwalk` + `rayon` saturate
   the SSD on a cold scan; cold-scanning `~` on a laptop
   finishes in seconds where `du -sh ~/*` is a coffee break.
2. **Mark-then-delete in the TUI.** `m` marks an entry, `M`
   marks all visible, `Ctrl-r` deletes everything marked after
   one confirmation. No "select, exit, `rm -rf`, re-scan" loop
   — the TUI updates its size totals in place after the delete.
3. **One-shot mode is not an afterthought.** `dua a b c` is a
   first-class subcommand that prints a sorted ranking with no
   TUI overhead, so it slots into shell pipes and CI guards
   the same way `du -sh` does. Most TUI-first tools force you
   into the TUI for any output.

For an LLM-CLI workflow, `dua` is the right tool to feed an
agent a "what is on disk" snapshot before asking it to plan a
cleanup — `dua --format bytes -t 100MB ~ -A` emits a list of
every entry over 100 MB, which packs straight into
[`files-to-prompt`](../files-to-prompt/) as cleanup context.

## Vs Already Cataloged

- **Vs [`ncdu`](../ncdu/):** `ncdu` is the C/Zig classic — modal
  ("scan now, browse the snapshot") with a single-threaded
  scanner. `dua` parallelizes the scan and unifies one-shot and
  interactive modes in one binary. Pick `ncdu` for the smallest
  dependency on a remote box, `dua` for laptop-grade speed and
  scriptable aggregates.
- **Vs [`gdu`](../gdu/):** `gdu` is the Go answer — also
  parallel, also interactive. Roughly comparable performance;
  `dua` has the cleaner one-shot aggregate command and
  byte/block/apparent format flags, `gdu` has a longer-standing
  TUI. Try both, pick by ergonomics.
- **Vs [`dust`](../dust/):** `dust` is "tree-view, colored
  bars, no TUI" — print-and-exit. Use `dust` when you want a
  pretty `du -h --max-depth=N` replacement; use `dua` when you
  want to *act* on the result interactively.
- **Vs [`duf`](../duf/):** `duf` is `df` (mounted-filesystem
  capacity), not `du` (per-directory size). Orthogonal — `duf`
  answers "how full is the disk," `dua` answers "what is using
  the disk."

## Caveats

- **No remote-mount awareness past `--stay-on-filesystem`.**
  Cross-mount aggregation works but the parallel scanner can
  hammer NFS / SMB shares; pass `-x` (a.k.a.
  `--stay-on-filesystem`) when scanning `/` or `~` on a host
  with network mounts.
- **Delete is permanent.** `Ctrl-r` does not move to a
  trash directory — it `unlink(2)`s. Pair with a snapshot
  filesystem (APFS, ZFS, btrfs) or use [`trash-cli`] equivalents
  if you want recoverable deletes.
- **TUI markings are session-local.** Exit the TUI and the
  marked set is gone; there is no "save a cleanup plan, review,
  apply" workflow. If you need that, drop to one-shot mode and
  pipe through `xargs rm -rf` after review.
- **Hard links double-count by default.** `dua` shows the size
  charged to *each path*; passing `-l` toggles "count hard
  links once" if you care about the underlying inode footprint
  (e.g., on a homebrew prefix full of hard-linked Cellar
  entries).
