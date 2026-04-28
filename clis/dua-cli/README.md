# dua-cli

- **Repo:** https://github.com/Byron/dua-cli
- **Version:** v2.34.0 (latest stable, 2025)
- **License:** MIT ([LICENSE](https://github.com/Byron/dua-cli/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install dua-cli` · `cargo install dua-cli` · `pacman -S dua-cli` · prebuilt binaries on the GitHub release page · binary name is `dua`

## What it does

`dua` (Disk Usage Analyzer) is a fast `du` replacement with a built-in
interactive TUI for exploring and reclaiming disk space.

- Walks a directory tree in parallel using all available cores, returning
  a sorted size summary in seconds where `du -sh *` would take minutes.
- `dua i` opens an interactive browser: drill into subdirectories with
  arrow keys, sort by size / count / mtime, mark entries with `m`, and
  delete them with `Ctrl-R` (with a confirmation prompt).
- Reports apparent size, on-disk size, or hard-link-aware size; handles
  sparse files and symlinks correctly.
- Honours a deny-list of paths (e.g. skip `/proc`, `/sys`, network
  mounts) and respects byte-vs-block accounting per filesystem.
- Pure read-only by default — destructive deletion only happens through
  the explicit interactive `Ctrl-R` flow, never as a flag on the
  non-interactive `dua` command.

## When to use it

Reach for `dua` when you need to answer "what's eating my disk?" on a
laptop, a build server, or a container host and the answer needs to
arrive in one screenful, not after `du | sort | head` finishes. The
interactive mode is the right tool when you've identified a culprit
directory (an old `~/Library/Caches`, a stale `node_modules`, a forgotten
Docker volume mount) and want to walk into it, mark a few subtrees, and
delete them in one session without leaving the terminal. It pairs well
with [`kondo`](../kondo/) (which targets project-build artefacts
specifically) as the generic "any subtree" complement.

## When NOT to use it

Skip it when you need a non-interactive size report you'll feed into a
pipeline — `du -sh` or `find ... -size` are still simpler for scripts.
Skip it for **remote** or networked storage where the TUI's snappy feel
disappears (latency makes interactive nav painful; use `du` over `ssh`
and grep). Skip it when the disk pressure is on a system you can't
afford to delete from interactively (production hosts, shared servers)
— a read-only inventory tool plus a reviewed deletion script is safer.
And skip it for filesystems where apparent vs allocated size genuinely
matters for accounting — read the docs and pick the flag deliberately,
or use a filesystem-native tool (`zfs list`, `btrfs filesystem du`).
