# rip2

> **Safe, ergonomic `rm` replacement that moves files to a
> graveyard you can `unrip`** — a Rust rewrite and actively
> maintained fork of nivekuil's original `rip` (now archived).
> Files go to `$XDG_DATA_HOME/graveyard` (configurable), survive
> across sessions, and can be restored in reverse-chronological
> order with one command. Pinned to **v0.9.4** (SPDX: `BSD-3-Clause`,
> [LICENSE](https://github.com/MilesCranmer/rip2/blob/master/LICENSE)).

Source: <https://github.com/MilesCranmer/rip2>

## TL;DR

`rip2` is what you set up the day after you `rm -rf`'d the
wrong directory. It behaves like `rm` (single command, accepts
globs, recurses by default for directories) but instead of
unlinking it *moves* the inode to a graveyard tree that mirrors
the original path. `rip -u` (unbury) restores the most recently
ripped item; `rip -s` (seance) lists everything currently in
the graveyard. Crucially: this is a maintained fork — upstream
`rip` was archived in 2024, `rip2` is the live continuation.

## Install

```bash
# Homebrew (macOS / Linux)
brew install rip2

# Cargo
cargo install --locked rip2

# Nix
nix-env -iA nixpkgs.rip2

# Pre-built release binary
curl -Lo rip2.tar.gz "https://github.com/MilesCranmer/rip2/releases/download/v0.9.4/rip-aarch64-apple-darwin.tar.gz"
tar xf rip2.tar.gz
sudo install rip /usr/local/bin/

# verify
rip --version    # rip 0.9.4
```

The binary is named `rip` (not `rip2`) — the `2` is only in the
project / package name to disambiguate from the archived
upstream.

## License

BSD-3-Clause — see
[LICENSE](https://github.com/MilesCranmer/rip2/blob/master/LICENSE).

## One Concrete Example

```bash
# 1. rip a file (goes to graveyard, NOT deleted)
rip notes.txt

# 2. rip a directory recursively
rip ./build/

# 3. list everything in the graveyard
rip -s

# 4. unbury the most recently ripped item back to its original path
rip -u

# 5. unbury a specific path pattern
rip -u ~/Projects/notes.txt

# 6. inspect what would be ripped without doing it
rip --inspect huge-dir/

# 7. permanently empty the graveyard (this IS irreversible)
rip -s | xargs rm -rf   # or: rm -rf "$(rip --graveyard)"

# 8. point the graveyard somewhere with more space
export GRAVEYARD=/var/tmp/$USER-graveyard
rip giant-log.txt
```

## Niche It Fills

**Reversible deletion at the shell, with zero ceremony.**
Sits between `rm` (instant, irreversible) and `trash-cli` (uses
the desktop trash spec, varies by OS). `rip2` is a single
self-contained Rust binary with its own graveyard, so it works
identically on a headless server, a macOS laptop, and a Linux
desktop — no FreeDesktop trash spec required.

## Why use it

1. **Restorable by default.** `rip foo` then `rip -u` brings
   `foo` back to its original path. `rm` cannot.
2. **Path-preserving graveyard.** The graveyard mirrors the
   original directory tree, so two files both named `config.toml`
   from different projects do not collide.
3. **`-s` (seance) shows the graveyard.** Audit what is reclaimable
   before you free the disk.
4. **Single static Rust binary.** No Python, no `gio trash`
   dependency, no D-Bus. Works on a server.
5. **Actively maintained.** Upstream `nivekuil/rip` was archived
   in 2024; `rip2` is the live fork with bug fixes (move-across-
   filesystems, large-file handling, Windows support).

## Vs Already Cataloged

- **Vs `rm` (POSIX):** `rm` is irreversible and that is the
  point on production systems. `rip` is for interactive shells
  where "I will probably want this back in 30 seconds" is the
  common case.
- **Vs [`fd`](../fd/) + `rm`:** orthogonal — `fd` finds files,
  `rip` removes them safely. Pipe one into the other:
  `fd -e .log -X rip`.

## Caveats

- **The graveyard grows forever.** Nothing auto-empties it.
  Periodically `rm -rf "$(rip --graveyard)"` or set up a cron
  to trim by mtime.
- **Cross-filesystem rips are copy+delete.** Ripping a 50 GB
  file from `/data` to a graveyard on `/home` will copy 50 GB.
  Set `GRAVEYARD` to a path on the same filesystem as your
  working tree.
- **Not a substitute for `git`.** For source files you actually
  care about, `git` (or `jj`) is still the right backstop.
  `rip` catches accidents on un-tracked files: build artefacts,
  exports, downloads, scratch directories.
- **Binary is `rip`, not `rip2`.** If you have any other tool
  on `$PATH` named `rip` (some old audio-rip tools), shadow
  carefully.
