# git-spice

- **Repository:** https://github.com/abhinav/git-spice
- **Latest version:** v0.27.0
- **License:** GPL-3.0 — verified at [`LICENSE`](https://github.com/abhinav/git-spice/blob/main/LICENSE) (raw: https://raw.githubusercontent.com/abhinav/git-spice/main/LICENSE)
- **Niche:** **Stacked-diff** workflow on top of plain `git` and
  GitHub / GitLab — keep a chain of small, dependent branches in
  sync, submit them as a stack of small reviewable PRs, restack
  cleanly when the base moves, all from one `gs` CLI.

## What it does

`git-spice` (`gs`) tracks parent/child relationships between local
branches and gives you primitives to navigate, restack, and submit
the whole stack. The mental model is a *tree of branches rooted at
`main`*, where each branch is one self-contained change that depends
on the changes above it.

```
# Initialise tracking in a repo (asks which branch is the trunk)
gs repo init

# Create a new branch on top of the current one and commit
gs branch create feat/parser     # = git checkout -b + tracked
git commit -am "parser scaffold"
gs branch create feat/parser-impl
git commit -am "implement parser"
gs branch create feat/parser-tests
git commit -am "tests"

# See the stack
gs log short

# Navigate
gs up           # parent ↑   (gs down to go child ↓)
gs top / gs bottom

# Trunk moved underneath you — restack the whole tower in one command
gs stack restack

# Submit the whole stack as N stacked PRs (one per branch), each
# targeting its parent branch. Re-runs update existing PRs.
gs stack submit
```

## Why interesting

Most teams cap PR size by gentleman's agreement ("keep it under 400
LoC"), then routinely break their own rule because the alternative
— *manually maintaining* a chain of dependent branches in `git` — is
genuinely awful. The pain points are well known:

1. **Navigation:** `git checkout` by branch name across a 6-deep
   stack is busywork; you forget which branch is which.
2. **Restacking:** when `main` moves (or branch 2 of 6 gets
   review feedback), you need to rebase every descendant in the
   right order without losing track. `git rebase --onto` chains are
   error-prone and one conflict mid-chain leaves the stack
   half-rebased.
3. **Submission:** GitHub's UI assumes one PR = one branch off
   `main`. Submitting six dependent PRs with the right base
   branches, the right "this depends on #123" comments, and updating
   them all when you push, is mostly clerical work.

`git-spice` collapses all three. `gs up` / `gs down` / `gs top` /
`gs bottom` make the stack feel like a linked list. `gs stack
restack` rebases every branch in dependency order in one command;
mid-chain conflicts pause for resolution and resume from where they
stopped (`gs rebase continue`). `gs stack submit` opens the right N
PRs against the right N base branches and includes a stack-aware
navigation comment, then on the next push updates them in place
instead of forcing you to close and re-open.

It is a **pure local layer over `git`** — no daemon, no server, no
custom remote. The state lives in a `refs/spice/data/...` ref in
your repo, so it travels with the worktree and a teammate without
`gs` installed sees normal branches and normal PRs. That makes it
viable to adopt unilaterally without org-wide coordination, which
is the practical difference between "stacked diffs are great in
theory" and "we actually use them".

## Pairs well with

- [`git-branchless`](../git-branchless/) — overlapping niche
  (smartlog + restack). `git-branchless` leans more on the
  "everything is a commit, branches are optional" Mercurial-shaped
  model; `git-spice` keeps named branches as the unit and is
  organised around "submit this stack as PRs".
- [`gh`](../gh/) / [`glab`](../glab/) — `gs` calls them under the
  hood for PR creation; you'll still use them for the rest of your
  PR-day workflow (review, merge, checks).
- [`jj`](../jj/) — if you can adopt a different VCS frontend over
  the same `git` repo, `jj` gives you the cleanest restack story in
  the catalog. `git-spice` is the answer when "stay in `git`,
  unchanged, no team migration" is a hard constraint.
- [`git-absorb`](../git-absorb/) — solves the *intra-branch* version
  of the same problem (auto-amend a fixup into the right historical
  commit); pairs naturally with `gs` for "address PR comments on
  branch 3 of 6 without disturbing branches 1, 2, 4, 5, 6".

## When to skip

- Solo project, no PRs, no review process — stacked diffs solve a
  *review* problem; if there is no reviewer, the overhead of
  `gs branch create` over `git checkout -b` buys you nothing.
- Your team is already on Phabricator / Sapling / Graphite and the
  workflow is working — `gs` is the answer for "stay on plain
  GitHub + plain `git`", not a reason to migrate off a working
  stacked-diff stack.
- You routinely ship one large PR per feature and reviewers prefer
  it that way — stacked diffs change the *unit* of review; if the
  team genuinely wants the bigger unit, the tooling won't help.
