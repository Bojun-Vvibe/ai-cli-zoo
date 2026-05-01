# sapling

## What it does
A scalable, git-compatible source control system whose CLI binary is `sl`. Every working-copy edit is automatically a commit-in-progress (no separate `add` / `commit` cycle), commit hashes stay stable across rewrites via a `commit cloud` sync layer, and the included Interactive Smartlog (`sl web`) renders the commit graph as a live-reloading browser UI where you drag-rebase, edit messages, and split commits with the mouse. `sl` clones an existing GitHub repo (`sl clone https://github.com/owner/repo`), pushes back as a normal git remote, and lets each developer adopt the workflow individually without forcing the team off git.

## Why it's interesting
The interesting story isn't "another VCS" — it's that this is the source-control system that powers a single trillion-line monorepo at one of the largest engineering orgs on the planet, open-sourced and rebuilt to interoperate with git so individuals can adopt it without coordination. Three concrete wins over `git` for daily work: (1) the *smartlog* (`sl ssl`) shows only your draft stack and the public branches it depends on (no more `git log --all --graph` wall of irrelevant branches); (2) `sl absorb` automatically routes uncommitted changes back into the matching commit in your stack instead of one fixup commit per chunk; (3) `sl undo` is a real, transactional undo across rebases, amends, and splits backed by an operation log — `git reflog` reconstruction is replaced with `sl undo` / `sl redo`. Pair with the [Interactive Smartlog VS Code extension](https://marketplace.visualstudio.com/items/meta.sapling-scm) and the merge / rebase loop becomes drag-and-drop. Conflict-free interop with git means a teammate on plain `git push` never knows you're using `sl`.

## Niche category
Git-compatible scalable source control with stack-first UX and operation-log undo — the [`jj`](../jj/) sibling shipped from a giant-monorepo lineage

## Repo
https://github.com/facebook/sapling

## Version pinned
`0.2.20260317-201835+0234c21f`

## License
- SPDX: `MIT` (also `GPL-2.0-only` for some components inherited from the Mercurial lineage; check `LICENSE` for the per-tree breakdown)
- License file in upstream repo: `LICENSE`

## Install
```sh
# macOS (Homebrew)
brew install sapling

# Pre-built Linux x86_64 portable tarball
curl -L https://github.com/facebook/sapling/releases/download/0.2.20260317-201835%2B0234c21f/sapling-0.2.20260317-201835+0234c21f-linux-x64.tar.xz | tar xJ
sudo install sapling/sl /usr/local/bin/

# Linux arm64
curl -L https://github.com/facebook/sapling/releases/download/0.2.20260317-201835%2B0234c21f/sapling-0.2.20260317-201835+0234c21f-linux-arm64.tar.xz | tar xJ
sudo install sapling/sl /usr/local/bin/

# Verify
sl --version
```

## Usage
```sh
# Clone a github repo with sapling — same URL as git
sl clone https://github.com/owner/repo
cd repo

# Smartlog: the only commit graph view you'll need 90% of the time
sl ssl                          # CLI smartlog (your draft stack + public branches)
sl web                          # browser-based Interactive Smartlog at http://localhost:3011

# Edit a file — sl auto-tracks it as a "pending commit"
echo 'fix' >> src/lib.rs
sl status                       # shows draft commit with the change, no `add` step
sl commit -m "fix bug"          # finalize / name the commit

# Stack a follow-up
echo 'tests' >> src/lib_test.rs
sl commit -m "tests for fix"
sl ssl                          # see the 2-commit stack on top of public main

# Absorb uncommitted edits into the matching commits in the stack automatically
echo 'tweak' >> src/lib.rs
sl absorb                       # routes the diff back into "fix bug", no fixup commit

# Move around the stack interactively
sl prev / sl next               # walk down/up the stack, working copy follows
sl goto <hash-or-bookmark>      # equivalent of git checkout

# Rebase the whole stack onto latest main
sl pull
sl rebase -d main

# Real undo — including across rebases, amends, splits
sl undo                         # reverts the last repo-modifying op
sl redo

# Push back to the github remote as a normal git push
sl push --to remote/main        # or `sl pr submit` to open a PR via the gh integration
```

## When to pick `sapling` vs alternatives
- **vs plain `git`**: stay on git when you collaborate with a small team that all uses git CLI as-is and you don't feel pain in the rebase / amend / log loop. Switch (per-individual; no team coordination needed) to sapling when stack-based development, smartlog visualisation, and transactional undo would speed up your daily loop.
- **vs [`jj`](../jj/)**: the closest peer — both reframe git around stable change-IDs and a real undo log. jj is younger, more aggressively redesigning the model (conflicts as first-class commit content, op-log as a graph), and has rapid release cadence. sapling is older, comes from a giant-monorepo production lineage, and ships a polished browser-based Interactive Smartlog out of the box. Pick jj when you want the cleanest current-day model and the most active community; pick sapling when the visual smartlog and the absorb / split / hide workflow are the killer features for you, or when you want the option to one-day adopt the EdenFS virtual file system that lets sapling scale to multi-million-file checkouts.
- **vs [`git-spice`](../git-spice/) / [`git-branchless`](../git-branchless/) / [`git-machete`](../git-machete/)**: those bolt stack-based UX onto git itself without changing the underlying VCS. Pick those when a team-wide VCS swap is off the table; pick sapling when you can change the binary you personally type but still need to push to the team's git remote.
- **vs Mercurial**: sapling is the spiritual successor to the Mercurial-derived in-house VCS the same lineage built. Plain Mercurial is no longer the right pick for new work; sapling is.
