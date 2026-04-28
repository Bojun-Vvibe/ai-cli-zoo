# git-town

- **Repo:** https://github.com/git-town/git-town
- **Version:** v22.7.1 (latest stable, 2026)
- **License:** MIT ([LICENSE](https://github.com/git-town/git-town/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install git-town` · `scoop install git-town` · prebuilt binaries on the GitHub release page · `go install github.com/git-town/git-town/v22@latest` · binary name is `git-town` (every command is also exposed as a `git` subcommand, e.g. `git town hack`, `git town sync`)

## What it does

`git-town` is a high-level porcelain on top of `git` that automates
the boring bookkeeping of trunk-based workflows with **stacked
branches**. The mental model: every feature branch has a declared
parent (usually `main`, but it can be another feature branch),
and `git town` knows that parent. From that one fact the tool
derives every multi-step branch operation:

- `git town hack <name>` — cut a fresh feature branch from the
  perimeter (main / master / development), syncing first.
- `git town append <name>` — stack a new branch on top of the
  current branch (parent = current branch, not main).
- `git town sync` — for *every* local branch in your stack: pull
  its parent, rebase or merge the parent into it, push, and
  recurse down the stack so the whole tree stays current.
- `git town propose` — open a PR for the current branch with the
  *correct base branch* (the parent), not main.
- `git town ship` — squash-merge the current branch into its
  parent and clean up local + remote refs across the whole stack.

It supports rebase, merge, fast-forward, and compress sync
strategies, and remembers per-repo configuration in `.git-branches.toml`.

## When to pick it / when not to

Reach for `git-town` when your team works in **stacked PRs**
(branch B depends on branch A, both open at the same time, both
need to stay rebased on `main`). It is also the right tool when
you have ≥3 long-running feature branches at any time and you keep
forgetting which one is the parent of which, or when you regularly
need to split a too-big branch into a stack and want one command
that re-parents the children.

Skip it for solo work on one branch at a time — `git checkout -b` +
`git rebase main` is enough. Skip it if your team uses long-lived
release branches with custom merge strategies that don't map onto
"branch has a parent." Skip it if you have moved to
[`jj`](../jj/) — Jujutsu's anonymous-branch model makes most of
git-town's value redundant. For tools that *also* manage stacked
PRs but as a hosted product, see Graphite or `gh stack`.

## Why it matters in an AI-native workflow

Coding agents naturally produce stacked work: a `claude-code`
session that adds a feature often emits "first refactor X, then
add Y on top of the refactor." Reviewers want those as two
PRs that land in order, not one giant PR. Without `git-town`, the
agent (or the human) has to remember every branch's parent,
manually rebase children when the parent changes after review
feedback, and update PR base branches after each `ship`. With
`git-town`, the agent runs `git town append <child>` to stack a
follow-up, and a single `git town sync` after the parent gets
review fixes cascades the rebase through every descendant
branch — cutting a class of "I forgot to rebase the child PR"
mistakes that bite agentic workflows hardest.

## Example invocations

```bash
# One-time setup per repo (interactive prompts for main branch, hosting, sync strategy)
git town config setup

# Cut a fresh feature branch from main, syncing main first
git town hack my-feature

# Stack a follow-up branch on top of the current branch
git town append my-feature-part-2

# Sync every branch in the current stack (pull main, rebase parents into children)
git town sync --all

# Open a PR for the current branch with the correct base branch
git town propose

# Squash-merge current branch into its parent and clean up refs
git town ship

# Re-parent the current branch (move it from old parent to new)
git town set-parent

# Visualise the local branch hierarchy
git town branch

# Undo the last git-town command (works for hack/sync/ship/etc.)
git town undo
```

## Alternatives in this catalog

- [`jj`](../jj/) — Jujutsu treats every commit as anonymous and
  rebasable, making stacked work the default; pick `jj` if you
  can adopt a new VCS, pick `git-town` if you must stay on `git`.
- [`git-branchless`](../git-branchless/) — overlapping ergonomics
  (smartlog, restack, hide); `git-branchless` is more about the
  *commit graph*, `git-town` is more about the *branch / PR*
  workflow on top of it.
- [`lazygit`](../lazygit/) and [`gitui`](../gitui/) — TUIs that
  make manual rebasing tolerable; `git-town` removes the manual
  rebasing entirely for stacked workflows.
- [`git-absorb`](../git-absorb/) — pairs nicely: use `git-town`
  for branch topology, `git-absorb` for in-branch fixup hygiene.
