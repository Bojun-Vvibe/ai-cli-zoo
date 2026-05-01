# gh-poi

## What it does
A **`gh` CLI extension that safely deletes merged / closed local
branches** by cross-checking them against the GitHub API. Walks every
local branch, asks GitHub whether the associated PR is `merged` or
`closed`, classifies each branch as **deletable** / **not-deletable** /
**unknown**, prints a colored preview tree with the PR number + state
next to each branch, asks for confirmation, then runs `git branch -D`
on the safe subset and (optionally) prunes the matching remote
tracking refs. Handles the awkward case where your branch was
squash-merged (so `git branch --merged` does **not** see it as
merged) by trusting the PR state instead of the commit graph.
The `--scan deep` flag in v0.17.0 walks PRs across *all* configured
remotes (forks + upstream) instead of just `origin`, catching the
"I PR'd from my fork, then forgot about the local branch" case.

## Why it's interesting
Different shape from `git branch --merged | xargs git branch -d`
(misses squash-merges, misses rebase-merges, blows up on detached
HEAD), from `git-trim` (Rust, similar idea but no PR-state lookup —
purely commit-graph based, so same squash-merge blind spot), from
`git-branchless smartlog` (different problem — visualizes the stack,
doesn't reap), from manual deletion in the GitHub web UI (slow, only
deletes the *remote* branch, leaves the local one), and from
`gh pr list --state merged --json headRefName` + custom shell
scripting (you end up writing this tool yourself). gh-poi is the
*PR-aware local-branch reaper* shape: pick it specifically when you
work in a squash-merge repo and your `git branch` output has slowly
grown to 60+ stale entries you're afraid to delete by hand. Do
**not** pick it for non-GitHub remotes (GitLab / Gitea / Forgejo —
no PR API integration), for cleaning up *remote* branches at scale
(use the GitHub web UI's branch-deletion bulk action, or a server-
side workflow), or for repos where you intentionally keep
long-lived feature branches around as backups.

## Niche category
PR-aware local-branch janitor — `gh` extension that classifies and
deletes local branches by querying GitHub PR state (handles
squash-merge correctly).

## Repo
https://github.com/seachicken/gh-poi

## Version pinned
`v0.17.0` (latest tagged release, published 2026-04-18)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# As a gh extension (recommended — `gh` >= 2.0 required)
gh extension install seachicken/gh-poi

# Upgrade later
gh extension upgrade poi

# Homebrew (standalone binary, not gh-attached)
brew install seachicken/tap/gh-poi

# Prebuilt binaries (Linux / macOS / Windows / FreeBSD)
# https://github.com/seachicken/gh-poi/releases/tag/v0.17.0
curl -L -o gh-poi \
  https://github.com/seachicken/gh-poi/releases/download/v0.17.0/darwin-arm64
chmod +x gh-poi && sudo mv gh-poi /usr/local/bin/
```

## Usage examples
```sh
# Dry-run preview: show which branches would be deleted, with PR state
gh poi --dry-run

# Real run: prompt, then delete merged + closed branches locally
gh poi

# Skip the confirmation prompt (use in scripts / cron)
gh poi --dry-run=false

# Deep scan: walk PRs from all remotes (fork + upstream), not just origin
gh poi --scan deep

# Inspect what gh-poi sees for a specific repo (debug auth / token scope)
gh poi --debug
```
