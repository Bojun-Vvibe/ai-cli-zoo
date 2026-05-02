# fdupes

> **The classic Adrián López C-implemented duplicate-file finder
> that walks one or more directory trees, groups files by size,
> short-circuits non-matches with a partial-content hash, and
> confirms duplicates with a full byte-for-byte comparison so the
> "these are the same file" answer is exact (not "the SHA-256
> happened to collide", though that is also vanishingly
> unlikely) — output is plain stdout, one duplicate set per
> blank-line-separated block, ready to pipe into `xargs rm`.**
> Pinned to **v2.3.0** (as of 2025),
> [LICENSE](https://github.com/adrianlopezroche/fdupes/blob/master/LICENSE),
> MIT.

Source: <https://github.com/adrianlopezroche/fdupes>

## TL;DR

`fdupes` is the original Unix-shaped duplicate finder: point it
at a tree (`fdupes -r ~/Downloads`) and it prints groups of
identical files. Flags cover the practical workflow:
`-r` recursive, `-S` show file size, `-n` skip empty files,
`-A` exclude hidden, `-d` interactive delete with a per-group
keep-which prompt, `-N` (with `-d`) preserve the first match and
delete the rest non-interactively, `-L` hardlink duplicates
together (reclaim disk without losing any path), `-s` follow
symlinks. The byte-by-byte final compare means a hash collision
cannot delete the wrong file — paranoia-grade safety for the
"de-dupe my photo backup" workflow.

## Install

```bash
# Homebrew (macOS / Linux)
brew install fdupes

# Debian / Ubuntu
sudo apt install fdupes

# Fedora / RHEL
sudo dnf install fdupes

# Arch
sudo pacman -S fdupes

# From source
git clone https://github.com/adrianlopezroche/fdupes.git
cd fdupes && make && sudo make install

# Run
fdupes -r -S ~/Downloads        # recursive scan, show sizes
fdupes -r -d -N old_backups/    # delete dupes, keep first match, no prompt
```

## License

[MIT](https://github.com/adrianlopezroche/fdupes/blob/master/LICENSE),
SPDX `MIT`.

## Niche / positioning

Pick `fdupes` over [`fclones`](../fclones/) when the workload is
small-to-medium (a few hundred GB, a few hundred thousand files)
and the goal is the simplest, most-shipped-by-distros tool —
fclones wins on million-file trees with its parallel walk + Rust
core + reflink-aware dedup, fdupes wins on "it is already
installed on every Linux box on the planet". Pick over
[`czkawka`](../czkawka/) when the workflow is shell-pipe-shaped
not GUI-shaped — czkawka has a Slint GUI and a CLI, fdupes is
CLI-only and composes naturally with `xargs` / `awk` / `find`.
Pick over [`rdfind`](https://github.com/pauldreik/rdfind) and
`jdupes` when familiarity matters more than micro-optimisations
(jdupes is a fdupes fork with extra features and is a reasonable
swap; rdfind takes a different ranking-then-acting approach).
Skip when the dedup target is *block-level* on a CoW filesystem
(use `duperemove` on btrfs / `bees` for online dedup) — fdupes
operates at file granularity and its `-L` hardlink mode is the
file-level equivalent.
