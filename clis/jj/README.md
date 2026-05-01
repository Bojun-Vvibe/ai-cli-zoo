# jj (Jujutsu)

## What it does
Git-compatible version control system that replaces the working-copy-plus-index-plus-commit model with a single "every change is already a commit" model: the working copy is itself a mutable revision that gets automatically amended on every command, branches are anonymous by default, and conflicts are first-class objects that live inside commits instead of blocking the operation. Backs onto a real `.git` repo so you can use `jj` and `git` interchangeably on the same checkout.

## Why it's interesting
Orthogonal to git-the-CLI without throwing away the git ecosystem: operations like `jj rebase`, `jj split`, `jj squash`, and `jj undo` are atomic and never leave you in a half-finished MERGING / REBASING state, and the operation log (`jj op log`) makes any change reversible by id. Pairs well with stacked-PR workflows because anonymous branches plus easy history rewriting let you reshape a stack without per-branch bookkeeping.

## Niche category
Version control — git-compatible alternative VCS frontend

## Repo
https://github.com/jj-vcs/jj

## Version pinned
`v0.40.0`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
brew install jj
# or
cargo install --locked --bin jj jj-cli
```

## Usage examples
```sh
# Clone an existing git repo with a colocated git backend
jj git clone https://github.com/owner/repo.git && cd repo

# Start a new change on top of main and edit files
jj new main -m "wip: refactor parser"
$EDITOR src/parser.rs

# Split the current change in half interactively
jj split

# Undo the last operation (any operation, including a bad rebase)
jj op log
jj undo
```
