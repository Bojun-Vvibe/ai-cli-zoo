# tig

> **The original ncurses TUI for git** — a single C binary that
> wraps `git log`, `git diff`, `git blame`, `git stash`, `git
> reflog`, and the index/staging UI into one keyboard-driven
> repository browser. Pinned to **tig-2.6.0**
> ([COPYING](https://github.com/jonas/tig/blob/master/COPYING),
> GPL-2.0).

Source: <https://github.com/jonas/tig>

## Category

Git TUI. Sits beside the modern Rust entrants (`gitui`,
`lazygit`, `gitu`) — but predates all of them by ~15 years
(initial release 2006), is C/ncurses, ships in every distro,
and is what every other git TUI's keymap was eventually
benchmarked against. Read-only by default; staging/stash
actions are explicit keystrokes.

## What it does

`tig` opens directly into a view (default: log) and lets you
navigate commits, refs, diffs, blames, the reflog, and the
staging area entirely with the keyboard. Every view is a
filtered window over the same underlying git plumbing —
arrow keys move, `<Enter>` drills in, `q` pops back. It can
also act as a pager (`git log | tig`) so you can pipe any git
command's output into a navigable view, including non-standard
ones like `git log --oneline -- some/path`.

## Why catalog-worthy

1. **Universally installed.** `tig` is in every package
   manager Bojun's likely to encounter — Homebrew, apt, pacman,
   FreeBSD ports, Alpine. When you SSH into a build box and
   need to triage a regression on commit history, `tig` is
   the answer that's already there.
2. **Pager mode is unique.** `git log --grep=foo --all | tig`
   turns any ad-hoc git query into a navigable view with
   blame/diff drill-down. The Rust TUIs are full apps; tig
   is also a filter.
3. **Stable keymap.** The keymap has barely changed since
   the early 2010s; muscle memory transfers across decades
   and machines.
4. **2.6.0 actually ships improvements.** Despite its age,
   the 2025-09-16 release added blame-without-working-tree
   support, hide-`+/-`-signs in diff view, a committer column,
   and Unicode 17 via utf8proc 2.11.0.

## Install

```bash
# Homebrew
brew install tig

# apt
sudo apt-get install tig

# Verified release tarball (2.6.0, sha256 in release notes)
curl -L https://github.com/jonas/tig/releases/download/tig-2.6.0/tig-2.6.0.tar.gz -o tig-2.6.0.tar.gz
curl -L https://github.com/jonas/tig/releases/download/tig-2.6.0/tig-2.6.0.tar.gz.sha256 -o tig-2.6.0.tar.gz.sha256
shasum -a 256 -c tig-2.6.0.tar.gz.sha256
tar xf tig-2.6.0.tar.gz && cd tig-2.6.0 && make prefix=/usr/local install

# Verify
tig --version    # tig version 2.6.0
```

## License

GPL-2.0 — see
[COPYING](https://github.com/jonas/tig/blob/master/COPYING).
Copyleft; binaries are fine to install and use anywhere, but
modifications you redistribute must stay GPL-2.0.

## One Concrete Example

```bash
# default: open log view of HEAD
tig

# log view scoped to one path, follow renames
tig -- src/lib.rs

# blame at HEAD on one file — drill into the commit at any line
tig blame src/lib.rs

# pager mode: feed any git output into navigable tig
git log --grep='fix.*panic' --all --since='3 months ago' | tig

# show only the staging area + working tree (interactive add/reset)
tig status

# reflog view — invaluable after an "oh no I rebased over X"
tig reflog
```

Inside any view: `j/k` move, `<Enter>` opens detail split,
`d` toggles diff, `t` tree view at this commit, `b` blame at
this line, `R` refresh, `:set diff-options=-w` ignore-whitespace
on the fly, `q` pop, `Q` quit.

## Niche It Fills

The Rust TUIs (`gitui`, `lazygit`, `gitu`) are excellent and
have richer write-side workflows (interactive rebase UIs,
inline commit editing). `tig` wins on *read* workflows and
*ubiquity*: there is no machine — local, remote, container,
recovery shell — where it's not one package install away,
and its pager mode (`<any git command> | tig`) has no
equivalent in the Rust TUIs. For "I need to understand this
repo's history *right now* on a box I just SSH'd into", tig
is still the lowest-friction answer 19 years in.
