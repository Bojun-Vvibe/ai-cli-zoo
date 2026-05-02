# stow

> **A symlink farm manager for dotfiles and source-built software.**
> Take a directory tree, "stow" it into a target, and `stow`
> creates the matching symlinks. Unstow undoes them. The whole
> tool is a careful answer to "how do I install many packages
> into one prefix without their files trampling each other".
> Pinned to **v2.4.1**
> ([COPYING](https://git.savannah.gnu.org/cgit/stow.git/tree/COPYING),
> GPL-3.0-or-later).

Source: <https://www.gnu.org/software/stow/> (git:
<https://git.savannah.gnu.org/cgit/stow.git>)

## TL;DR

`stow` reads a *package directory* (e.g. `~/dotfiles/zsh/`) and
mirrors its tree into a *target directory* (e.g. `~`) by creating
symlinks: `~/dotfiles/zsh/.zshrc` becomes `~/.zshrc → ../dotfiles/zsh/.zshrc`.
It is folding-aware: if a whole subdir doesn't conflict, it
symlinks the *directory* (one link); if any sibling needs to be
written into that subdir later, it transparently *un-folds* —
replaces the dir-link with a real dir of file-links — so two
packages can share a parent.

The result: dotfiles and source-built `/usr/local` installs live
in one git repo as plain trees, and `stow <pkg>` / `stow -D <pkg>`
install / uninstall them atomically without copying.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install stow

# Debian / Ubuntu
apt install stow

# Arch
pacman -S stow

# Fedora
dnf install stow

# from source (Perl, no compile step)
git clone https://git.savannah.gnu.org/git/stow.git
cd stow && autoreconf -iv && ./configure && make && sudo make install

# verify
stow --version          # stow (GNU Stow) version 2.4.1
```

## License

GPL-3.0-or-later — see
[COPYING](https://git.savannah.gnu.org/cgit/stow.git/tree/COPYING).
Strong copyleft. `stow` is invoked as a CLI from your scripts and
generates symlinks; the GPL applies to *modifications of stow
itself*, not to the dotfiles you stow. Safe to use as a tool in
any environment; only think about the licence if you fork the
Perl source.

## One Concrete Example

```bash
# Layout
~/dotfiles/
├── zsh/
│   ├── .zshrc
│   └── .zshenv
├── git/
│   └── .gitconfig
├── nvim/
│   └── .config/
│       └── nvim/
│           └── init.lua
└── ssh/
    └── .ssh/
        └── config

# 1. Stow the zsh package into $HOME (default target = parent of cwd)
cd ~/dotfiles
stow zsh
# now: ~/.zshrc -> dotfiles/zsh/.zshrc
#      ~/.zshenv -> dotfiles/zsh/.zshenv

# 2. Stow several packages at once
stow zsh git nvim ssh
# ~/.gitconfig, ~/.config/nvim/init.lua, ~/.ssh/config all
# become symlinks back into the repo.

# 3. Dry-run before touching the filesystem
stow -nv nvim
# -n = no-op, -v = verbose: prints "LINK: .config/nvim/init.lua => ..."
# without making any changes. ALWAYS run this on a new package.

# 4. Un-stow (remove the symlinks but leave the package dir alone)
stow -D zsh
# ~/.zshrc and ~/.zshenv are removed; ~/dotfiles/zsh/ untouched.

# 5. Re-stow (unstow + stow, useful after editing .stow-local-ignore)
stow -R nvim

# 6. Stow source-built software into /usr/local without trampling
cd /usr/local/stow
sudo wget -O - https://example.com/foo-1.2.tar.gz | tar xz
cd foo-1.2 && ./configure --prefix=/usr/local/stow/foo-1.2 \
    && make && sudo make install
cd /usr/local/stow && sudo stow foo-1.2
# /usr/local/bin/foo, /usr/local/share/man/man1/foo.1, etc. are
# symlinks into /usr/local/stow/foo-1.2/. Removing the package is
# `sudo stow -D foo-1.2 && sudo rm -rf foo-1.2` — no `make uninstall`
# guesswork.
```

## Niche It Fills

**The "version-controlled dotfiles + zero magic" gap.** The
canonical dotfiles workflow is: keep one git repo, symlink the
files into `$HOME`, commit and push. The naive shell-script
approach (`ln -sf` in a loop) breaks the moment two packages
want to write into the same `~/.config/foo/` subdir, or when you
want to remove just one package's symlinks. `stow` is the answer
that has been solving exactly this since 1996: pure Perl, no
runtime config, no daemon, no JSON manifest — just "the layout
of the package dir IS the install plan". For source-built
software in `/usr/local/stow/<pkg>-<version>/`, the same
mechanism gives clean install/upgrade/uninstall without a
package manager.

## Why use it

Three things `stow` does that the obvious alternatives don't:

1. **Folding.** The cleverest part. If `~/.config/nvim/` doesn't
   exist yet and only the `nvim` package writes into it, `stow`
   creates *one* symlink `~/.config/nvim → dotfiles/nvim/.config/nvim`.
   Later when you stow a `nvim-extras` package that also wants
   `~/.config/nvim/lua/extras.lua`, `stow` automatically
   *unfolds*: replaces the dir-link with a real dir, links the
   files individually, then folds again on `stow -D`.
2. **Conflict detection up-front.** `stow -nv pkg` reports every
   would-be conflict (existing file, existing wrong-target
   symlink, would-overwrite) before any change. Idempotent and
   safe to run twice.
3. **No state file.** The "what is installed" answer is
   recoverable purely from `find ~ -lname '*/dotfiles/*'`. There
   is no `~/.stowdb` to lose, no metadata to keep in sync, no
   "stow forgot what it did". The filesystem is the database.

For an LLM-CLI workflow, `stow` is the **dotfiles bootstrap
primitive**: an agent on a fresh machine can `git clone
~/dotfiles && cd ~/dotfiles && stow */` and the entire shell
environment (zsh, nvim, tmux, ssh, gpg) is wired up in two
commands, with a guaranteed clean uninstall if anything goes
wrong.

## Vs Already Cataloged

- **Vs [`chezmoi`](../chezmoi/):** `chezmoi` is a much larger tool
  that templates dotfiles, encrypts secrets, runs scripts, and
  syncs across machines via its own state. `stow` is just
  symlinks. Pick `chezmoi` when files differ per-machine or
  contain secrets; pick `stow` when you want one canonical tree
  and zero templating.
- **Vs [`yadm`](../yadm/):** `yadm` is a git wrapper that turns
  `$HOME` itself into a working tree. No symlinks. `stow`
  separates source (the package dir) from target (`$HOME`) via
  symlinks, which makes `git status` in `$HOME` empty. Pick
  `yadm` if you want git's full toolset against `$HOME`; pick
  `stow` if you want the source dir to be a normal repo
  reviewable on its own.
- **Vs a hand-rolled `ln -sf` install script:** Works for ten
  files; falls apart at the first shared subdirectory or when
  you want to remove one package's links. `stow`'s folding /
  unfolding logic is the part you would otherwise have to
  reinvent (badly).
- **Vs distro packages for source-built software:** A real
  `.deb` / `.rpm` / `apk` is more correct (dependencies, scripts,
  signing). `stow` is the cheap escape hatch when you just need
  `./configure --prefix=/usr/local/stow/foo-1.2 && make install
  && stow foo-1.2` to get out the door without packaging.

## Caveats

- **Symlinks only.** Programs that `stat()` and refuse symlinked
  config (rare but exists — some old SSH versions complain about
  symlinked `~/.ssh/config` permissions, some setuid binaries
  refuse to follow symlinks for security) need a copy-based tool
  instead.
- **Target defaults to parent-of-cwd.** Running `stow zsh` from
  `~/dotfiles/` targets `~`. From `~/foo/dotfiles/` it targets
  `~/foo/`. Either always pass `-t ~` explicitly, or always run
  from a directory whose parent is the intended target. Many
  first-time bug reports trace to this default.
- **No templating, no per-host logic.** Same files everywhere.
  Combine with `.stow-local-ignore` (per-package skip-list) for
  coarse "don't link this on this machine" control, or graduate
  to `chezmoi`.
- **Perl runtime required.** Tiny dependency in practice (Perl
  is on every Unix), but a relevant detail for absolutely
  minimal container images.
- **Permissions are inherited from the source files.** If the
  symlinked-to file in your repo is `0644`, the program reading
  the symlink sees `0644`. SSH config requiring `0600` is the
  classic example — set the mode in the source repo, not after.
