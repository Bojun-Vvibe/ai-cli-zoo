# git-machete

- **Repo:** https://github.com/VirtusLab/git-machete
- **Version:** v3.36.1 (2026 stable line)
- **License:** MIT ([LICENSE](https://github.com/VirtusLab/git-machete/blob/develop/LICENSE))
- **Language:** Python
- **Install:** `brew install git-machete` · `pipx install git-machete` · `pip install git-machete` · `snap install git-machete` · prebuilt packages on the GitHub releases page · invoked as `git machete …` (or `git-machete`)

## What it does

`git-machete` is a **branch-organizer** for repos where you have
many in-flight feature branches stacked on top of each other or
on top of `main`. It maintains a small `.git/machete` file that
declares the parent/child relationship between branches, then
exposes a single command — `git machete status` — that draws the
whole forest, marks which branches are in sync with their parent,
which need a rebase, and which have an open PR. From that view,
`git machete traverse` walks the tree branch-by-branch and asks
you exactly one question per branch ("rebase onto parent?",
"push?", "create PR?"), so updating ten stacked branches after
`main` moves becomes a guided five-minute pass instead of an
afternoon of `git rebase --onto` math.

It also handles the parts that are easy to get wrong manually:
detecting that a branch was squash-merged into its parent (so it
can be removed from the tree), proposing the right `--onto` for
each rebase, syncing PR metadata with GitHub/GitLab/Bitbucket
via `git machete github`/`gitlab`/`bitbucket` subcommands, and
reordering the tree with `git machete slide-out` when an
intermediate branch is no longer needed. The tree is plain text,
so it diffs and commits cleanly if you want to share a stack
layout across machines.

## Example

```bash
# discover branches and propose a tree
git machete discover

# walk the tree, rebasing+pushing each branch interactively
git machete traverse
```

## When to pick it / when not to

Reach for `git-machete` if you regularly stack PRs (feature on
top of refactor on top of cleanup) and feel the pain every time
`main` advances. It is the most mature OSS option for tree-shaped
branch maintenance and does not require a server-side component.
If you only ever have one feature branch off `main`, plain `git
rebase` is fine. If you want server-rendered stacked-PR UIs,
look at Graphite or `gh stack`; `git-machete` deliberately keeps
state in your local repo and treats the forge as a sync target,
not a source of truth.
