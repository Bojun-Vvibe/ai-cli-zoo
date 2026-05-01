# spr

## What it does
A **stacked-PR workflow tool** in Rust that maps each local commit on
your feature branch to a separate GitHub PR, while preserving the
linear stack relationship between them. Run `spr diff` and every
commit since `main` becomes (or updates) its own PR with a
`spr/<author>/<branch>/<n>` head ref; the tool injects a
machine-readable header into each commit message so it can find the
PR it owns on the next run. `spr land` merges the *bottom* PR of the
stack and rebases the rest cleanly. `spr amend` lets you target a
specific commit in the stack for `git commit --fixup`-style edits
without rewriting the whole branch. Authenticates via GitHub device
flow (v1.3.7), no `~/.netrc` rituals.

## Why it's interesting
Different shape from `git push -f` of one giant feature branch
(reviewers drown in a 40-file PR, you can't land anything until the
whole thing is approved), from `gh pr create` (one PR per branch —
no stacking abstraction, you manage the rebase chain by hand), from
[`git-spice`](../git-spice/) (also stacked PRs, but branch-per-change
instead of commit-per-change — Go binary, separate metadata file
under `.git/spice` instead of trailers in commit messages), from
[`git-branchless`](../git-branchless/) (broader rewrite/restack
toolkit, not GitHub-PR-aware on its own), from Graphite / Sapling's
hosted stacked-PR UX (cloud service / different VCS), and from
[`git-machete`](../git-machete/) (visualizes + restacks
branch-per-change stacks, doesn't author the PRs for you). spr is
the *one-commit-equals-one-PR, GitHub-only, Rust binary* shape:
pick it specifically when your team has agreed that small reviewable
units are commits (not branches) and you want a tool that round-trips
the stack to GitHub PRs without leaving the terminal. Do **not** pick
it for non-GitHub remotes (no GitLab / Gitea support), for teams that
prefer branch-per-change stacking (use `git-spice` or `git-machete`),
or for solo repos where one PR per branch is already fine.

## Niche category
Stacked-PR tool — one local commit ↔ one GitHub PR, with a Rust binary
that owns the rebase + push + PR-update dance via commit-message trailers.

## Repo
https://github.com/spacedentist/spr

## Version pinned
`v1.3.7` (latest tagged release, published 2025-08-25)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install spr

# Cargo (any platform)
cargo install spr

# Nix
nix profile install nixpkgs#spr

# From source
git clone https://github.com/spacedentist/spr && cd spr && cargo build --release
```

## Usage examples
```sh
# One-time: authenticate spr against GitHub (device flow, v1.3.7+)
spr init

# Push every commit on the current branch as its own PR (creates or updates)
spr diff

# Same, but mark them ready-for-review instead of draft
spr diff --update-message --reviewers alice,bob

# Land the bottom PR of the stack, rebase the rest onto main
spr land

# Edit the 3rd commit of the stack without rewriting commits 4..N
git commit --fixup HEAD~2 && spr amend

# Show what spr thinks the current stack is, before you push
spr list
```
