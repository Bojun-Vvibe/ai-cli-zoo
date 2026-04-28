# git-absorb

- **Repo:** https://github.com/tummychow/git-absorb
- **Version:** 0.9.0 (latest stable, 2025)
- **License:** BSD-3-Clause ([LICENSE.md](https://github.com/tummychow/git-absorb/blob/master/LICENSE.md))
- **Language:** Rust
- **Install:** `brew install git-absorb` · `cargo install git-absorb` · `pacman -S git-absorb` · prebuilt binaries on the GitHub release page · binary name is `git-absorb` (invoked as `git absorb`)

## What it does

`git-absorb` is a Mercurial-style `hg absorb` port for Git. You stage
hunks that fix bugs in earlier commits on your branch, then run
`git absorb`, and it walks each staged hunk, finds the most recent
commit in your branch that last touched those lines, and creates a
`fixup!` commit pointing at that commit. A subsequent
`git rebase -i --autosquash` (or `git absorb --and-rebase`) folds the
fixups into their target commits silently. The end result: a
clean, logically-grouped branch where every commit stands on its
own, without you having to manually run `git commit --fixup=<sha>`
for each hunk and hand-pick the right SHA.

## When to pick it / when not to

Reach for `git-absorb` when you are partway through a feature branch
of 5–20 commits, code review (or your own re-read) reveals small
mistakes spread across several of those commits, and you want each
fix to land in the commit that introduced the bug rather than as a
trailing "fix typo" / "address review" commit. It is the right tool
when your branch will be reviewed commit-by-commit (Gerrit, stacked
diffs, patch-series workflows) or merged with `--ff-only` /
rebase-merge so each commit must be coherent on its own.

Skip it when you squash-merge everything anyway — the per-commit
hygiene buys you nothing. Skip it for hunks that *don't* belong to
any single prior commit (a hunk that crosses three commits' worth
of changes will be silently dropped from absorption; `git-absorb`
will report it as unassigned and you fix it by hand). Skip it on
the trunk branch and on any commit that is already published to a
shared remote — `git-absorb` rewrites history.

## Why it matters in an AI-native workflow

When an agent iterates on a branch — implement, test, fix, test,
fix — it tends to produce a long tail of micro-fixup commits whose
diff context belongs back in the original implementation commit.
That noise survives review and bisect: `git bisect` lands on a
"WIP" commit that doesn't even build, code review has to mentally
fold "fix off-by-one in helper" into "add helper", and the eventual
merge commit's history is unreadable. `git-absorb` lets the agent
stage a fixup, run one command, and have the change land in the
right ancestor commit — preserving "one commit = one logical
change" without asking the agent to track SHAs by hand. It composes
cleanly with [`jj`](../jj/) users (who get this by default) and
gives `git`-only repos the same ergonomics.

## Example invocations

```bash
# Stage just the fixes you want absorbed; leave unrelated work unstaged
git add -p

# Dry-run: show which staged hunk would be absorbed into which commit
git absorb --dry-run

# Absorb: create one fixup! commit per hunk, targeting the right ancestor
git absorb

# Absorb and immediately autosquash-rebase to fold the fixups in
git absorb --and-rebase

# Limit the search to the last N commits (default: walks until merge-base)
git absorb --base origin/main

# Force-attach a hunk that touches lines never modified on this branch
# (by default such hunks are reported as unassigned and skipped)
git absorb --force

# Typical end-to-end flow on a feature branch:
#   git rebase -i origin/main      # tidy the branch first
#   <work, review, find mistakes>
#   git add -p                      # stage only the fixes
#   git absorb --and-rebase --base origin/main
#   git push --force-with-lease
```

## Alternatives in this catalog

- [`jj`](../jj/) — Jujutsu, a Git-compatible VCS where this kind of
  retroactive edit is the default model (every change is mutable
  until pushed); pick `jj` if you can adopt a new VCS, pick
  `git-absorb` if you must stay on `git`.
- [`lazygit`](../lazygit/) and [`gitui`](../gitui/) — interactive
  TUIs that can stage hunks and craft fixup commits manually; they
  give you the *staging* surface but not the automatic
  hunk-to-commit assignment.
- [`gitu`](../gitu/) — Magit-style Git porcelain in the terminal;
  similar manual-fixup workflow as `lazygit`.
- [`delta`](../delta/) — pairs well as the diff viewer when you are
  reviewing what `git absorb --dry-run` is about to do.
