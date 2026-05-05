# diskus

> **A minimal, parallel `du -sh` rewrite that returns the size
> of one directory, fast** — single number on stdout, no tree,
> no per-file breakdown, no flags to memorise. Walks the
> directory tree across all CPU cores in parallel and uses the
> raw `stat` byte count (or apparent size, on demand). On a
> warm cache against a 1M-file tree, it routinely beats `du
> -sh` by 5–10×. Single static Rust binary (~1 MB). Pinned to
> **v0.9.0** (SPDX: `Apache-2.0`,
> [LICENSE-APACHE](https://github.com/sharkdp/diskus/blob/master/LICENSE-APACHE)
> / [LICENSE-MIT](https://github.com/sharkdp/diskus/blob/master/LICENSE-MIT)
> dual-licensed).

Source: <https://github.com/sharkdp/diskus>

## TL;DR

`diskus path/` returns the total size of `path/` and nothing
else. The whole UI is one positional argument, an optional
`--apparent-size` for "what `ls -l` reports" instead of the
allocated block count, `-B / --block-size` for human / SI / raw
units, and `-j / --threads` to override the rayon parallel
walker. Unlike `du`, there is no tree-rendering path; unlike
[`dust`](../dust/), there is no per-subdir bar chart; unlike
[`gdu`](../gdu/) / [`ncdu`](../ncdu/) / [`dua`](../dua/), there
is no interactive TUI. The whole point is "I want one number,
right now, in the smallest possible binary, parallelised across
my cores" — the 90% case that the bigger tools' richer UIs slow
down.

By the same author as [`fd`](../fd/),
[`bat`](../bat/), [`hyperfine`](../hyperfine/),
[`hexyl`](../hexyl/), and [`pastel`](../pastel/) — same
single-purpose-Rust-binary aesthetic, same
benchmark-driven-design discipline.

## Install

```bash
# Homebrew
brew install diskus

# Cargo
cargo install diskus --locked

# Arch
pacman -S diskus

# Debian / Ubuntu (releases)
curl -LO https://github.com/sharkdp/diskus/releases/download/v0.9.0/diskus_0.9.0_amd64.deb
sudo dpkg -i diskus_0.9.0_amd64.deb
```

## Usage

```bash
# Total size of cwd
diskus

# Total size of a specific path
diskus ~/Downloads

# Apparent size (matches ls -l, ignores filesystem block padding)
diskus --apparent-size /var/log

# Override parallelism
diskus -j 16 /mnt/bigdisk

# In a script — raw bytes for arithmetic
SIZE_BYTES=$(diskus -B 1 ~/cache)

# Compare two directories quickly
diskus dir-a; diskus dir-b
```

## Why it matters

The `du -sh path/` shape is the most-typed disk-usage command
on any UNIX box, and on a modern NVMe + 16-core laptop the
single-threaded GNU `du` leaves 90% of the hardware idle while
you wait. `diskus` exploits that — parallel walk + minimal
output — and gets the answer back in the time it takes to
glance away from the terminal. It is deliberately *not* a
disk-usage explorer (use [`gdu`](../gdu/) /
[`ncdu`](../ncdu/) / [`dua`](../dua/) /
[`dust`](../dust/) for tree breakdown / interactive cleanup);
it is the right tool when the question is exactly "how big is
this directory" and the answer is exactly one number — in
shell scripts, in CI quota checks, in pre-deploy size gates,
in `watch -n 5 diskus /tmp` while a build runs. Pairs with the
explorers (use `diskus` to discover the offender, switch to
`gdu` / `dust` to drill in) and with [`hyperfine`](../hyperfine/)
(the same author's benchmark tool — useful for proving the
speedup on your own filesystem before adopting).
