# jujutsu

> **Git-compatible version control system that reshapes the
> commit graph as a first-class operation** — every working
> copy is *itself* a commit, every command rewrites history
> safely via the operation log, and conflicts are stored *in*
> the commit graph rather than blocking your shell. Pinned to
> **v0.40.0**
> ([LICENSE](https://github.com/jj-vcs/jj/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/jj-vcs/jj>

## TL;DR

`jj` (jujutsu) is a from-scratch VCS that speaks the on-disk
git format as a backend (so `jj git clone` / `jj git push`
interoperate with any git remote — GitHub, Gitea, plain SSH
bare repos, your existing CI), but the user-facing model is
deliberately *not* git. The two pivots: (1) **the working
copy is a commit**. There is no staging area; every edit you
save is automatically amended into the current revision (`@`),
and `jj new` opens a fresh empty child whenever you want a
boundary. (2) **history rewriting is the default operation,
not a recovery tool**. `jj rebase`, `jj squash`, `jj split`,
`jj absorb`, `jj describe`, `jj edit <rev>` all work on any
revision in the graph — including ones in the middle of long
stacks of dependent changes — and `jj` automatically rebases
descendants. Every operation is recorded in `jj op log`, so
"undo the last five rewrites" is `jj op restore <op>` and
genuinely safe. Conflicts become first-class objects: a
rebase that hits a conflict produces a *commit* with conflict
markers stored in its tree, you keep working on top of it,
and resolving it later is just another edit to that commit.
The CLI is `clap`-driven Rust, `--help` is comprehensive,
revsets (a Mercurial-style query language for commits) make
expressions like `jj log -r 'mine() & ~empty() &
description(glob:"feat:*")'` ergonomic.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install jj

# Cargo (any platform with Rust 1.84+)
cargo install --locked --bin jj jj-cli

# Nix
nix profile install nixpkgs#jujutsu

# Arch
pacman -S jujutsu

# Single-binary download (GitHub releases, v0.40.0)
curl -L -o jj.tar.gz \
  https://github.com/jj-vcs/jj/releases/download/v0.40.0/jj-v0.40.0-x86_64-unknown-linux-musl.tar.gz
tar xzf jj.tar.gz && sudo mv jj /usr/local/bin/

# macOS arm64
curl -L -o jj.tar.gz \
  https://github.com/jj-vcs/jj/releases/download/v0.40.0/jj-v0.40.0-aarch64-apple-darwin.tar.gz
tar xzf jj.tar.gz && sudo mv jj /usr/local/bin/

# Build from source pinned to release tag
git clone --depth 1 --branch v0.40.0 https://github.com/jj-vcs/jj.git
cd jj && cargo install --locked --path cli
```

## Usage

```bash
# Clone an existing git repo into a colocated jj+git checkout
jj git clone --colocate https://github.com/owner/repo.git
cd repo

# Look at the graph (revsets work in -r)
jj log -r 'all()'                  # full graph
jj log -r 'mine() & ~empty()'      # only my non-empty changes
jj log -r 'trunk()..@'             # everything between trunk and working copy

# Make a change. There is no `git add`.
echo "hi" >> README.md
jj describe -m "docs: add greeting"   # set the message of @
jj new                                 # start a fresh empty commit on top

# Reorder / split / squash arbitrary commits in your stack
jj rebase -r abc123 -d def456          # move one commit
jj squash --from abc123 --into def456  # fold abc123 into def456
jj split -r abc123                     # split a commit interactively
jj absorb                              # auto-distribute working changes into ancestors

# Push to git remote (creates one branch per change you mark with `jj bookmark`)
jj bookmark create my-feature -r @
jj git push --bookmark my-feature

# Undo anything — really anything — via the op log
jj op log
jj op restore <op-id>
```

## Why it's interesting

The "next-generation VCS" slot has been crowded for a decade
(`git` won, `hg`/Mercurial lost mindshare, `pijul` /
`fossil` / `bazaar` / `darcs` stayed niche), and the usual
trade is "novel model but no interop, so no team adopts it".
`jj` takes the opposite trade: **keep git as the wire
format, replace only the user model**. That single decision
unlocks adoption — you can drop `jj` into any git repo
today, your colleagues keep using `git`, your CI keeps
calling `git fetch`, and you never have to migrate the
remote. What you get on top is the model `git rebase -i` was
trying to be: stacked-PR workflows where you edit the
*middle* of a 12-commit stack with one command and the rest
auto-rebase, conflict resolution that doesn't trap you in a
mid-rebase shell, and a real undo log instead of `reflog`
spelunking. Pick `jj` when (a) you spend serious time in
stacked-diff workflows ([Phabricator](https://www.phacility.com/)/
Graphite/Sapling-style) and want native primitives instead of
`git rebase --interactive --autosquash` rituals, (b) you've
been bitten by losing work to a botched rebase and want
`op log` as a real safety net, or (c) you maintain a long-
lived feature branch that constantly needs to absorb
upstream — `jj absorb` + first-class conflicts make that
loop dramatically less painful. Not the right pick when the
team is git-only and won't tolerate a colocated `.jj/` dir,
or when your VCS server is a Mercurial / SVN install
(`jj`'s git backend is the only mature one — the
hand-written backend is an experimental future direction).
Active development by ex-Google VCS engineers; releases
cadence is roughly monthly with a clear deprecation policy
in `CHANGELOG.md`. Compare with [`gitu`](../gitu/) (terminal
UI *over* git itself, not a replacement) and
[`bit`](../bit/) (git wrapper that adds branch UX without
changing the model) — `jj` is the only entry in the slot
that replaces the model wholesale while keeping the wire
format.
