# diskonaut

## What it does
A terminal **disk space navigator** that walks a filesystem and renders the
result as an interactive **squarified treemap** in a TUI: each rectangle is a
file or directory, area is proportional to size on disk, colour distinguishes
directories from files, and arrow keys + Enter let you descend into the largest
boxes to find what is actually eating the volume. Unlike top-down summarisers
it scans incrementally and updates the treemap as it goes, so you can start
acting on a 2 TB disk after a few seconds instead of waiting for a full
walk to finish. `Backspace` ascends, `Tab` toggles between the treemap view
and a sorted list view, `Delete` removes the highlighted entry (with a
confirm prompt), and the bottom status bar shows total scanned, free space,
file count, and the current path. Single static Rust binary, no daemon, no
config file, no network.

## Why it's interesting
Different shape from `du -sh *` (text-only, no interaction, no incremental
output, scales poorly past ~100k files), `ncdu` (interactive curses list
sorted by size — fast, scriptable, the long-standing default — but list
shape, not spatial), `gdu` (Go reimplementation of ncdu with parallel
scanning — same list shape, faster on SSDs), `dust` (one-shot tree view in
your terminal — no navigation, prints and exits), `dua` / `dua-cli` (Rust
parallel scanner with both interactive and one-shot modes — list shape, very
fast — the catalog already covers it), and GUI tools like Baobab / DaisyDisk
/ WizTree (treemap shape, but desktop-only). diskonaut is the *treemap shape
in a terminal* option: pick it specifically when "I want to see the spatial
layout of disk usage over SSH" is the ask, because the treemap visually
clusters siblings inside their parent and makes the answer to "is this one
huge file or ten thousand small ones?" visible at a glance. Do **not** pick
it for scripting / CI / non-interactive size summaries (use `du`, `dust`,
or `dua` in batch mode), or for filesystems with millions of tiny files
where the rectangle for any individual file becomes invisibly small.

## Niche category
Interactive disk-usage navigator — squarified treemap TUI with incremental
scanning and in-place delete.

## Repo
https://github.com/imsnif/diskonaut

## Version pinned
`0.11.0` (latest tagged release, adds Windows support)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE.md`

## Install
```sh
# Homebrew
brew install diskonaut

# Cargo
cargo install diskonaut --locked

# Arch
sudo pacman -S diskonaut

# Debian / Ubuntu (recent releases)
sudo apt install diskonaut

# Pre-built tarballs:
# https://github.com/imsnif/diskonaut/releases/latest
```

## Usage examples
```sh
# Scan the current directory and open the treemap
diskonaut

# Scan a specific path (typically what you actually want)
diskonaut ~/Downloads
diskonaut /var/log
diskonaut /

# Inside the TUI:
#   ↑ ↓ ← →   move highlight between rectangles
#   Enter     descend into the highlighted directory
#   Backspace ascend to parent
#   Tab       toggle treemap ⇄ sorted list view
#   Delete    delete the highlighted file/dir (with confirm)
#   /         filter by name
#   q         quit

# Useful over SSH — no GUI required
ssh server -- 'diskonaut /var'

# Combine with sudo for system paths you cannot read as your user
sudo diskonaut /var/lib
```
