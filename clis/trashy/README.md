# trashy

> **A fast, XDG-spec-compliant `rm` replacement that puts files in the
> trash instead of unlinking them** — single Rust binary, drop-in
> for `rm`, with a `list / restore / empty` subcommand surface that
> the system file manager will also see, because it writes to the
> same `~/.local/share/Trash/{files,info}` layout that GNOME, KDE,
> Nautilus, and Dolphin use. Pinned to **v2.0.0** (commit
> `10d11a63aa542d847ca04860c629000c8f4a0f4e`,
> [LICENSE-APACHE](https://github.com/oberblastmeister/trashy/blob/master/LICENSE-APACHE),
> Apache-2.0; also dual-licensed MIT via `LICENSE-MIT`).

Source: <https://github.com/oberblastmeister/trashy>

## TL;DR

`rm -rf` has no undo. Every developer has, at least once, deleted
the wrong directory and discovered the hard way that the shell does
not have a recycle bin. `trashy` is the small, fast fix: a Rust
binary called `trash` that, instead of `unlink(2)`-ing, *moves* the
file into the freedesktop.org Trash directory and writes the
matching `.trashinfo` metadata so it can be restored later — by
`trash restore`, by `gio trash --restore`, or by clicking
"Restore" in the file manager. Compared to the older Python
`trash-cli`, `trashy` is one binary, an order of magnitude
faster on large directories, and ships with `fzf`-friendly
`list`/`restore` UIs.

## Install

```bash
# Cargo
cargo install trashy

# Homebrew
brew install trashy

# From source
git clone https://github.com/oberblastmeister/trashy
cd trashy
cargo build --release
./target/release/trash --version    # trash 2.0.0

# verify
trash --version
```

Make it your default `rm`:

```bash
# in ~/.zshrc or ~/.bashrc
alias rm='trash put'
```

(Some users prefer to keep `rm` raw and just train themselves to
type `trash`. Either works — `trashy` does not patch `rm`.)

Daily use:

```bash
trash put scratch/             # trashes the dir
trash list                     # show recently trashed entries
trash restore                  # interactive picker, restores to original path
trash empty                    # permanently delete everything in the trash
trash empty --all              # also empty per-mountpoint trashes
```

## Why it's worth a slot in the zoo

Every other "modern UNIX rewrite" (`bat` for `cat`, `eza` for
`ls`, `dust` for `du`) is about *reading* better. `trashy` is
the rare one about *deleting* better — it removes a real,
recurring, irreversible category of mistakes from your shell
history. It is also a clean example of "do the boring,
specified thing well": it implements the freedesktop Trash spec
faithfully, so the trash you create from the terminal is the
same trash your DE knows about. That interop is the whole
reason it beats `mv ./file ~/.trash/`.

## Where it sits

- vs `rm`: `rm` is fast and final. `trashy` is fast and
  reversible. The cost is one extra rename and one tiny
  metadata file per delete.
- vs `trash-cli` (Python): same spec, same on-disk layout,
  much slower (Python startup + per-file Python overhead).
  `trashy` is a drop-in upgrade and the trash directories
  are mutually compatible — you can switch back and forth.
- vs `rip` (`nivekuil/rip`): `rip` is the other Rust trash
  CLI. It is more aggressive about removing the freedesktop
  layer (writes its own graveyard format under
  `/tmp/graveyard-$USER`), so it does *not* interoperate
  with the GUI trash. Pick `trashy` if you want the file
  manager to see your deletes; pick `rip` if you want a
  per-shell graveyard.
- vs `gio trash` (GNOME): same spec, ships everywhere GNOME
  ships, but the CLI ergonomics are clunky (`gio trash
  --restore` needs a numeric index from `gio trash --list`).
  `trashy`'s interactive `restore` picker is the win.
- vs `mv ./x ~/.Trash/`: macOS-only, no metadata, no
  per-volume trash — restoring loses the original path.

## Footguns

- **It is not `rm`.** Disk space is not freed until you run
  `trash empty`. On a tight `/home` partition, "deleting"
  10 GB and being surprised that `df` did not move is the
  classic trap. Schedule a periodic `trash empty --older
  7d` if this matters.
- **Cross-filesystem deletes are not "instant".** The trash
  spec requires the file to live on the same volume it was
  deleted from, so `trashy` writes per-mountpoint trashes
  (`<mount>/.Trash-$UID/`). Trashing a 50 GB file on a
  different mount than `$HOME` triggers a *copy*, not a
  rename — and that takes real time.
- **Some mounts forbid the trash dir.** Read-only mounts,
  noexec mounts without write perms, or weird FUSE
  filesystems will reject `.Trash-$UID/` creation. `trashy`
  falls back to refusing the delete (correct), but that can
  surprise scripts.
- **`trash empty` is final.** There is no second-level
  trash. Once emptied, recovery is back to forensic tools.
- **It does not protect against `rm`.** Aliasing `rm=trash
  put` only helps interactive shells; `Makefile`s, scripts
  with `#!/bin/sh`, and anything calling `unlink(2)`
  directly bypass it entirely. Treat `trashy` as "safety
  net for human typing", not "system-wide undelete".
- **The v2 release reorganized the CLI.** Pre-v2 used `trash
  -l` / `trash -r`; v2 moved to subcommands (`trash list`,
  `trash restore`). Old shell aliases need updating.
