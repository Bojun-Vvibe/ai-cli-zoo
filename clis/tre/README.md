# tre

> **A `tree` alternative that respects
> `.gitignore` and prints editor-jump shortcuts
> for each entry** — same recursive directory
> view as classic `tree`, but skips
> version-control noise by default and tags
> each line with an alias so `0`, `1`, `2`, …
> opens the matching path in your `$EDITOR`.
> Pinned to **v0.4.0**
> ([LICENSE-MIT](https://github.com/dduan/tre/blob/main/LICENSE-MIT),
> MIT).

Source: <https://github.com/dduan/tre>

## TL;DR

`tre` is a Rust rewrite of the classic `tree`
command that defaults to the behaviour you
actually want when you point it at a project
directory: **gitignore-aware** (uses the same
`ignore` crate as [`ripgrep`](../ripgrep/), so
`.gitignore`, `.ignore`, `.git/info/exclude`,
and global excludes all apply), **hidden-file
suppressed** by default (`-a` to opt in), and
**editor-integrated** — pass `-e` (or set
`TRE_EDITOR`) and every printed entry gets a
short alias like `[0]`, `[1]`, `[2]`; running
`0` in your shell opens that file in the editor
you configured. The aliases are written to a
shell-specific shim (`~/.config/tre/aliases.sh`
sourced from your `.zshrc` / `.bashrc` / `.config/fish`),
so the lifecycle is "list → eyeball → jump"
without copy-pasting paths. Output supports the
same `-L <depth>`, `-d` (directories only), and
`-s` (file sizes) flags as `tree`, so existing
muscle memory mostly transfers, and rendering
is fast enough on a Linux kernel checkout to
beat `tree -I 'target|node_modules|.git'` by
not having to read those subtrees at all.

## Install

```bash
# Homebrew
brew install tre-command

# Cargo
cargo install tre-command

# Arch Linux (AUR)
paru -S tre-command

# prebuilt binaries
# https://github.com/dduan/tre/releases

# verify
tre --version    # tre 0.4.0
```

## Basic usage

```bash
# project tree, gitignore respected
tre

# limit depth, show sizes
tre -L 2 -s

# directories only
tre -d

# include hidden files (.env, .git/, etc.)
tre -a

# editor-jump mode: pick a number afterwards
# add `eval "$(tre -e)"` to your shell rc once
tre -e
0   # opens the first listed entry in $EDITOR

# combine with a path filter
tre src -L 3 -e
```

## When to choose

- **You point `tree` at a real project directory
  and the first 200 lines are
  `target/`, `node_modules/`, or `.git/`** —
  tre's gitignore default makes the output
  immediately readable without an `-I` mini
  language.
- **You want a "list, then jump" workflow without
  installing a TUI file manager** — `tre -e` is
  the shortest path between "show me the layout"
  and "open that file in vim" without leaving
  your existing shell.
- **You use [`ripgrep`](../ripgrep/) and want a
  matching directory view** — both share the
  `ignore` crate, so what `rg` searches is
  exactly what `tre` lists.

## When NOT to choose

- **You need a full file manager** — use
  [`yazi`](../yazi/), [`nnn`](../nnn/),
  [`broot`](../broot/), [`xplr`](../xplr/), or
  [`lf`](../lf/); tre is one-shot output, not
  interactive navigation.
- **You need rich entry metadata** (permissions,
  owner, mtime, file types) — use [`eza --tree`](../eza/),
  which is `ls`-shaped and carries every
  long-form column.
- **You're shipping a script that already
  consumes classic `tree -J` JSON** — tre's
  output shape is its own; pin a real `tree` for
  parser stability.

## Why it fits the zoo

Belongs to the "small Rust rewrite that fixes
the one ergonomic gap of a Unix classic" cluster
([`bat`](../bat/) → `cat`, [`fd-find`](../fd-find/) → `find`,
[`ripgrep`](../ripgrep/) → `grep`,
[`eza`](../eza/) → `ls`, [`dust`](../dust/) → `du`,
[`procs`](../procs/) → `ps`). tre is the
`tree`-shaped member of that family, with
gitignore + editor-jump as the two specific
gaps it closes.

## Upstream pointers

- Repo: <https://github.com/dduan/tre>
- Release notes: <https://github.com/dduan/tre/releases>
- License: [MIT](https://github.com/dduan/tre/blob/main/LICENSE-MIT)
- Maintainer: [@dduan](https://github.com/dduan)
