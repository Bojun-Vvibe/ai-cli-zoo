# eza

> **`ls(1)` clone with colours, icons, Git status, and a tree
> view** — a single Rust binary, the maintained successor to the
> archived `exa`. Pinned to **v0.23.4** (commit
> `5b7bc61765879cebc98613d65f7b99ceab9c6242`,
> [LICENCE.txt](https://github.com/eza-community/eza/blob/main/LICENCE.txt),
> EUPL-1.2).

Source: <https://github.com/eza-community/eza>

## TL;DR

`eza` is a drop-in `ls` with the long-listing format that GNU
coreutils never shipped: type-coloured names, human-readable
sizes by default, optional Nerd-Font icons per file type, a
`--git` column showing per-file `git status` flags, a `--tree`
mode that replaces `tree(1)`, and `--total-size` that recurses
into directories so `eza -lah --total-size` answers "where is
the disk going" without `du`. Same call shape as `ls`, same
flag letters where they overlap (`-l`, `-a`, `-h`, `-r`, `-S`,
`-t`), and stdout-is-not-TTY auto-detect drops colour for
pipes. The `exa` project was archived in 2023; `eza` is the
community-maintained fork that picked up where it left off.

## Install

```bash
# Homebrew (macOS / Linux)
brew install eza

# Cargo
cargo install --locked eza

# Linux package managers
# Arch: pacman -S eza
# Debian / Ubuntu (24.04+): apt install eza
# Fedora: dnf install eza
# Nix: nix-env -iA nixpkgs.eza

# Windows
# scoop install eza
# winget install eza-community.eza

# from a release tarball (any OS)
curl -Lo eza.tar.gz "https://github.com/eza-community/eza/releases/download/v0.23.4/eza_aarch64-apple-darwin.tar.gz"
tar xf eza.tar.gz
sudo install eza /usr/local/bin/

# verify
eza --version    # eza v0.23.4
```

For icons, install a Nerd Font (e.g. `brew install --cask
font-jetbrains-mono-nerd-font`) and call `eza --icons=auto`.
Drop the alias bomb in your shell rc:

```bash
alias ls='eza --group-directories-first'
alias ll='eza -lah --git --group-directories-first'
alias lt='eza --tree --level=2 --git-ignore'
```

## License

EUPL-1.2 — see
[LICENCE.txt](https://github.com/eza-community/eza/blob/main/LICENCE.txt).
The European Union Public Licence is a weak copyleft (similar
in spirit to LGPL / MPL) — distributing modified `eza` source
requires you publish your changes; bundling the unmodified
binary in a product does not. Practically irrelevant for
laptop / CI use.

## One Concrete Example

```bash
# 1. long view with human sizes, hidden files, Git status column
eza -lah --git
# Permissions Size User   Date Modified  Git Name
# .rw-r--r--  1.2k bojun  18 Apr 12:14    M README.md
# .rw-r--r--  430  bojun  18 Apr 12:10    -- .gitignore
# drwxr-xr-x   -   bojun  18 Apr 12:10    N- src/

# 2. tree view, depth 2, respects .gitignore
eza --tree --level=2 --git-ignore

# 3. directories first, sort by modified time, recent at top
eza -lah --group-directories-first --sort=modified --reverse

# 4. show recursive disk usage per top-level entry (cheap du)
eza -lah --total-size

# 5. icons on (requires Nerd Font in your terminal)
eza -lah --icons=auto --git

# 6. pipe-friendly: stdout is not a TTY, colour and icons drop
eza -1 src/ | xargs wc -l

# 7. only directories, sorted by entry count
eza -D --sort=size

# 8. one-liner sanity check before committing
eza -lah --git src/ tests/
```

## Niche It Fills

**The Git-aware `ls`.** When you are mid-edit in a repo and run
`ls`, the question you are usually answering is "what did I
touch and how big is it". `eza --git` answers both in one call
— per-file status flags (`M`, `N-`, `--`, `-I`, etc.) in a
dedicated column, sizes in human units, dirs vs files
distinguishable at a glance — without a separate `git status`
in another pane.

## Why use it

Three things `eza` does that GNU `ls` does not, that pay back
the switching cost:

1. **`--git` column shows per-file status inline.** Modified,
   new, ignored, untracked — each file in the long listing
   carries its `git status` flag in a fixed-width column. No
   second command, no diff against `HEAD` in your head.
2. **`--tree` replaces `tree(1)`.** Same colour / icon / size
   formatting as the long listing, with `--level=N` to limit
   depth and `--git-ignore` to skip what `.gitignore` excludes.
   One binary instead of `ls` + `tree` + `du`.
3. **`--total-size` recurses for directory size.** GNU `ls -lh`
   shows directory entries as `4.0K` (the inode block); `eza
   -lah --total-size` shows the recursive size of each top-level
   directory, turning `ls` into a quick `du -sh *` substitute.

The defaults — type colours from `LS_COLORS` (or `EZA_COLORS`),
human sizes, sane date format, group-directories-first when
asked — are the ones GNU coreutils never adopted because of
backwards compatibility. `eza` gets to start clean.

## Vs Already Cataloged

- **Vs [`yazi`](../yazi/):** orthogonal — `yazi` is an
  interactive TUI file manager (browse, preview, select, act);
  `eza` is a non-interactive lister for shell sessions and
  pipelines. Use `eza` when you just need to *see* the
  directory; reach for `yazi` when you need to *navigate* and
  act on selections.
- **Vs [`bat`](../bat/) / [`ripgrep`](../ripgrep/) /
  [`fd`](../fd/):** complementary — same Rust-rewrites-coreutils
  family, same "TTY-aware, ignore-aware, colour-by-default"
  ergonomics. A typical project shell uses all four:
  `ls`→`eza`, `cat`→`bat`, `grep`→`rg`, `find`→`fd`.
- **Vs `lsd` (not cataloged):** sibling — `lsd` is the other
  popular `ls` rewrite (also Rust, also colours + icons). `lsd`
  is closer to a pure visual upgrade; `eza` adds the `--git`
  column, `--total-size`, and `--git-ignore` tree mode that
  make it function as a workspace inspector, not just a
  prettier `ls`. If you want icons + colour and nothing else,
  `lsd` is fine; if you want Git awareness, pick `eza`.
- **Vs GNU `ls` (coreutils):** `ls` wins on portability (every
  Unix box has it, scripts depend on its exact output) and on
  shell-script safety (do not alias `ls=eza` in shared
  `/etc/profile` — scripts that parse `ls -l` output will
  break). `eza` wins on every interactive use.

## Caveats

- **Do not alias `ls=eza` system-wide.** Some shell scripts
  parse `ls -l` output by column position; `eza`'s columns are
  different. Alias in your *interactive* shell rc only
  (`~/.zshrc`, `~/.bashrc`), never in `/etc/profile.d/` or in
  `Dockerfile` `ENV`.
- **Icons require a Nerd Font in the terminal.** Without one,
  `--icons` renders as tofu (`􀀀` / `?` / unprintable). Install
  one from <https://www.nerdfonts.com/> first; no fallback for
  basic-Unicode-only terminals.
- **`--git` column adds a libgit2 walk per directory.** On
  giant repos (linux kernel scale) this adds noticeable
  latency; drop `--git` for those, or pin a smaller `--depth`
  on `--tree` invocations.
- **Date format is locale-dependent and uses your terminal
  width.** Narrow terminals truncate the date column; widen
  the terminal or pass `--time-style=long-iso` for a fixed
  shape.
- **EUPL-1.2 is unfamiliar to many corporate license scanners.**
  Practically equivalent to LGPL/MPL for binary use, but a
  scanner that does not know EUPL may flag the dependency. If
  your bill of materials must be on a known list, document it
  upfront.
- **`exa` is not `eza`.** Old tutorials and dotfiles refer to
  `exa`; that project was archived in 2023. Install `eza` and
  alias `exa=eza` if you need the old name; do not install
  `exa` from package managers that still ship the dead binary.
