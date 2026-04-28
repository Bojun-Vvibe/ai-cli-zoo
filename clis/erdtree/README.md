# erdtree

> **A modern `tree` that sums disk usage and respects `.gitignore`** —
> a single Rust binary (`et`) that walks a directory in parallel,
> renders a colored tree, and on every node prints its on-disk
> footprint (logical, physical, or block-count). Pinned to
> **v3.1.2**
> ([LICENSE](https://github.com/solidiquis/erdtree/blob/master/LICENSE),
> MIT).

Source: <https://github.com/solidiquis/erdtree>

## Category

Filesystem / disk-usage TUI. Sits between classic `tree`,
`du -sh *`, and `ncdu` — but it's a one-shot, scriptable binary,
not a full-screen interactive viewer.

## What it does

`et` walks a directory tree (in parallel via Rayon), sums the
size of every subtree, and prints a colored, sorted tree where
each line shows the path *and* its accumulated size. By default
it honors `.gitignore` and `.ignore`, hides dotfiles, and sorts
descendants by size descending — so the biggest offender is
always at the top of every level.

## Why it matters

If you have ever opened a project, run `du -sh *`, then `cd`'d
into the largest dir and re-run `du -sh *`, repeating until you
found the actual culprit — `et` collapses that whole loop into
one command. Unlike `du`, the output is a tree (so you keep
hierarchical context); unlike `tree`, it actually shows sizes;
unlike `ncdu`, it's a one-shot pipe-friendly command that fits
in a terminal screenshot or a script.

## Install

```bash
# Homebrew
brew install erdtree

# Cargo
cargo install --locked erdtree

# Arch
pacman -S erdtree

# Verify
et --version    # erdtree 3.1.2
```

## License

MIT — see
[LICENSE](https://github.com/solidiquis/erdtree/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# inside any project root — show top 10 levels, sorted by size,
# honoring .gitignore (default), human-readable sizes (default)
et -L 10

# only show directories, sorted by total size descending
et -L 10 --disk-usage logical -y inverted -t dir

# include hidden files and ignored files (everything on disk)
et -H -i

# pipe-friendly: emit JSON for downstream tools
et --output json | jq '.[] | select(.size > 100000000)'

# flat layout (no tree drawing), great for sorting in a script
et -y flat -L 3 | sort -k2 -h
```

## Niche It Fills

`tree` is read-only-pretty; `du` is size-only; `ncdu` is
interactive-only. `erdtree` is the only one that is *all of*:
fast (Rayon-parallel), tree-shaped, size-aware, gitignore-aware,
and one-shot scriptable. The killer feature in practice is
`-y inverted -t dir` — instantly answers "which directory in
this repo is eating my disk?" without an interactive viewer.
