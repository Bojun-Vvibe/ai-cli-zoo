# git-branchless

- **Repo:** https://github.com/arxanas/git-branchless
- **Version:** v0.10.0 (2024-10-14)
- **License:** Apache-2.0 / MIT dual ([LICENSE-APACHE](https://github.com/arxanas/git-branchless/blob/master/LICENSE-APACHE), [LICENSE-MIT](https://github.com/arxanas/git-branchless/blob/master/LICENSE-MIT))
- **Language:** Rust
- **Install:** `brew install git-branchless` · `cargo install --locked git-branchless` · prebuilt binaries on the GitHub release page · binary entry points include `git-branchless`, `git-smartlog` (alias `git sl`), `git-restack`, `git-record`, `git-undo`, etc.

## What it does

`git-branchless` is a suite of porcelain commands that adds
Mercurial / Jujutsu-style stacked-diff and "branchless" workflows on
top of plain git. It does not replace git — it augments it, and you
can drop it at any time without rewriting history.

- `git sl` (smartlog) renders a compact graph of your active
  commits — only the ones not yet upstream — instead of the wall of
  noise you get from `git log --all --graph`.
- `git move -s <src> -d <dest>` rebases an entire commit subtree onto
  another commit in one step, automatically restacking child
  branches and updating refs; this is the killer feature for stacked
  PR workflows where one commit at the bottom of the stack changes.
- `git undo` walks an event log of every ref/working-tree change and
  lets you interactively pick a prior state to restore — including
  reflog-invisible operations like `rebase --abort` aftermath.
- `git record` is an interactive commit builder (hunks, splits,
  amends) that competes with `git add -p` and `git commit --fixup`
  ergonomically.
- Uses an internal SQLite event log plus reachability analysis to
  track "visible" commits even when they have no branch attached,
  enabling truly branchless workflows where commits are first-class
  and branches are an afterthought.
- Coexists with normal `git` — every command is opt-in, the repo
  layout is unchanged, and `git fetch` / `git push` still work
  exactly as before.

## When to use it

Reach for it when you maintain stacks of dependent commits / PRs and
the daily friction is "I changed commit 3 of a 7-commit stack and
now I need to manually rebase commits 4–7 onto it" — `git move`
turns that into one command. Reach for it when you want
[`jj`](../jj/)-style ergonomics but cannot adopt jj's parallel
metadata model (because your tooling, hooks, or teammates assume
plain git). And `git undo` alone is a strong reason to install it
on any machine where you do non-trivial history surgery.

## When NOT to use it

Skip it for simple, linear feature-branch workflows — the smartlog
and event log are overkill if you do `checkout -b → commit → push →
PR → merge` and never rebase. Skip it if you've already moved to
[`jj`](../jj/) or [`gitoxide`](../gitoxide/)-based porcelain like
[`gitu`](../gitu/) — `jj` solves the same stacked-diff problem more
deeply, and running both layers on top of one repo is confusing.
Skip it on large monorepos where its initial event-log build can
take minutes on first run and feels heavy; benchmark on your repo
before committing your team to it. And skip it as a teaching tool
for git beginners — its mental model (visible commits, smartlog,
restacking) is a layer on top of git that assumes you already know
the underlying primitives.
