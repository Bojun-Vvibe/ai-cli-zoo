# jj

> **Jujutsu — a git-compatible VCS that fixes git's commit
> model** — every working-tree edit is automatically a commit
> (no `add` / `commit` cycle), the operation log is a first-class
> queryable history (`jj op log` lets you `jj op restore` any
> previous repo state including aborted rebases), conflicts are
> first-class commit content (a conflicted commit can be
> rebased, split, abandoned without resolving first), and
> change IDs (stable across rewrites) are separate from commit
> IDs (which change every rewrite) so `jj rebase` / `jj squash`
> / `jj split` don't break references. Backed by either a
> native Jujutsu store or a git store (`jj git init --colocate`)
> so a `jj`-managed repo is also a normal `git` repo every
> teammate can clone with `git`. Pinned to **v0.40.0** (commit
> `693bb0f5bb2b0ce546e0a47e7ed58416e9f0b19b`,
> [LICENSE](https://github.com/jj-vcs/jj/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/jj-vcs/jj>

## TL;DR

`jj` (Jujutsu) is a Rust-implemented VCS that keeps git's
storage compatibility but replaces git's *commit model* with
one that removes most of the cliff edges (detached HEAD,
"oops I committed to the wrong branch", "the rebase is now
broken and I lost my work", "I rewrote a commit and now my
collaborator's branch points to a non-existent SHA"). The core
bet: every working-tree state is *already* a commit
(automatically; no staging area, no `git add`), every
operation that changes the repo (commit, rebase, abandon,
restore, even `jj git push`) is recorded in an *operation log*
(`jj op log`) that you can `jj op restore <op-id>` to roll the
repo back to any previous state including aborted rebases,
conflicts are *content* of a commit (a `jj rebase` over a
conflicting change creates a "conflicted commit" you can
abandon / split / rebase further without ever resolving), and
identity has two layers: **change ID** (stable across all
rewrites — `kktrk...`, lowercase) and **commit ID** (changes
every rewrite — the SHA you'd see in `git log`). Branches
(`jj` calls them "bookmarks") are *pointers* you move
explicitly, not auto-following symrefs — your work is on a
nameless change by default and `jj bookmark set my-feature -r
@-` only when you're ready to publish. The `--colocate` mode
makes the same directory a fully valid `git` repo (`.git/`
directory + `.jj/` directory side-by-side), so a teammate on
plain git can clone, pull, push, and review PRs from your
branches without knowing `jj` exists.

## Install

```bash
# Homebrew (macOS / Linux)
brew install jj

# Cargo (latest from crates.io)
cargo install --locked --bin jj jj-cli

# pre-built binary from GitHub releases
curl -fsSL https://github.com/jj-vcs/jj/releases/latest/download/jj-v0.40.0-aarch64-apple-darwin.tar.gz | tar -xz
sudo install jj /usr/local/bin/

# init a new repo (native backend)
jj git init my-project && cd my-project

# OR clone an existing git repo into jj
jj git clone https://github.com/owner/repo.git
cd repo

# OR adopt jj inside an existing git repo (colocated — both .git/ and .jj/)
cd existing-git-repo
jj git init --colocate

# verify
jj --version    # jj 0.40.0
jj log          # see the change graph
```

The `--colocate` workflow is the recommended on-ramp for an
existing team — your collaborators keep using `git` against the
same `.git/` directory; you use `jj` for your local workflow,
and `jj git push` syncs your bookmarks to the git remote so
PRs work normally.

## License

Apache-2.0 — see [LICENSE](https://github.com/jj-vcs/jj/blob/main/LICENSE).
Permissive with patent grant (stronger than MIT for
multi-vendor settings); redistribute binaries freely.

## Hot keybinds / commands

`jj` is a CLI not a TUI, so these are the subcommand muscle
memory:

- `jj st` — show working-copy diff + parents (the `git status`
  equivalent; runs on every keystroke essentially since the
  working copy is always a commit)
- `jj log` — change graph (default reveals the recent work
  + bookmarks); `jj log -r 'mine() & ~immutable_heads()'`
  shows just your in-flight changes
- `jj new` — create a new empty change on top of the current
  one (the closest analogue to `git commit -m "" && checkout
  HEAD` — finalises the current change as immutable-ish, opens
  a fresh one for the next edit)
- `jj describe -m "msg"` — set the description of the current
  change (the `git commit -m` equivalent; can run any number
  of times — the description is metadata, not a snapshot
  trigger)
- `jj squash` / `jj squash -i` — fold the current change into
  its parent (`-i` opens an interactive picker for
  per-file / per-hunk choice — the `git commit --amend` +
  `git add -p` combo collapsed into one verb)
- `jj split` / `jj split -i` — break the current change into
  two changes (interactive picker for which hunks land in the
  first; the rest stay in a new commit on top)
- `jj rebase -d <change>` — rebase the current change onto
  `<change>`; conflicts become first-class commit content,
  the rebase always succeeds, you resolve at your leisure
- `jj abandon` — discard the current change (its descendants
  rebase automatically); recover with `jj op restore`
- `jj op log` — operation log (every repo-modifying operation
  recorded); `jj op restore <op-id>` rolls the entire repo
  back to that operation's state — the universal undo
- `jj bookmark set <name> -r @-` — create / move a bookmark
  to point at the parent of the working copy (the `git branch
  -f` equivalent — bookmarks don't auto-follow your edits)
- `jj git push -b <bookmark>` — push a bookmark to the git
  remote (in `--colocate` mode this writes to the same
  `.git/` your teammates use)
- `jj git fetch` — pull bookmarks from the git remote

## Why use it

Three things `jj` does that plain git does not, that change
the day-to-day shape of working with version control:

1. **The operation log is a universal undo.** Every
   repo-modifying command — commit, rebase, abandon, push,
   even checkouts — appends to `jj op log`. `jj op restore
   <op-id>` rolls the *entire repo* back to that operation's
   exact state, including the working-copy contents, branch
   pointers, and conflict resolutions. The "I broke a rebase
   30 commits in and I don't know how to recover" failure mode
   stops happening — you `jj op restore @-` and try again.
   Compare to `git reflog` which shows commit-level moves
   only, doesn't include the working tree, and doesn't restore
   branch pointers atomically.
2. **Conflicts as commit content, not interruption.** A
   `jj rebase` that hits a conflict *succeeds* — the
   resulting commit is "conflicted", visible in `jj log` with
   a marker, and can be rebased further, split, abandoned,
   or partially resolved without blocking other work. You
   resolve the conflict by editing the working copy when you
   want to (which produces a non-conflicted child commit).
   Compare to git, where a conflicted rebase halts mid-way,
   the working copy is in a special state, and other commands
   refuse until you `git rebase --continue` or `--abort`.
3. **Change IDs separate from commit IDs.** Every change has
   a stable **change ID** (`kktrk...`, lowercase, persists
   across rewrites) and a **commit ID** (the SHA, which
   changes every time you rewrite). Tooling that wants to
   reference a change across rewrites (review tools, stacked
   PRs, your own muscle memory) uses the change ID; storage
   and git interop use the commit ID. The "I rebased and
   force-pushed and now my colleague's branch points at a
   ghost commit" pattern goes away when both sides reference
   change IDs.

For an LLM-CLI workflow that produces a long stack of small
related commits (one per agent turn) that need to be reordered,
squashed, split, and rebased frequently, `jj`'s "every working
state is a commit" + "rebase always succeeds" + "operation log
undo" combination removes most of the friction. The
`--colocate` mode means you get this without leaving git, and
without forcing your team to switch tools.

## Vs Already Cataloged

- **Vs [`lazygit`](../lazygit/) / [`gitui`](../gitui/):**
  different layer. Those are *UIs on top of git*; `jj` is a
  *different VCS* (with git-storage compatibility). They don't
  compose directly — `lazygit` doesn't understand `jj`'s
  change-ID model or operation log. Pick `jj` when you want
  to change the underlying commit semantics; pick `lazygit`
  when you want a faster keymap on plain git.
- **Vs raw `git`:** `jj git init --colocate` makes a directory
  *both* a git repo and a jj repo — your local workflow uses
  `jj`, your team's workflow uses `git`, the `.git/` directory
  is the source of truth both sides see. Trade-off: the
  workflow vocabulary diverges from the rest of your team's
  (`jj squash` ≠ `git commit --amend`, `jj new` has no exact
  git analogue), so adoption is per-individual not per-team.
- **Vs `sapling` (Meta's git replacement, formerly Mercurial-
  derived):** sapling is the closest peer in design intent
  (also rewrites the commit model, also git-compatible). `jj`
  has stronger conflict-as-content semantics and a more
  developed operation log; `sapling` has stronger stacked-PR
  tooling (`sl ghstack`) and Meta's internal-tooling polish.
  Both Apache-2.0; both can adopt an existing git repo;
  picking between them is mostly community / docs preference.
- **Vs [`dbmate`](../dbmate/) / [`tach`](../tach/) / catalog
  CLIs in general:** orthogonal — `jj` is the VCS those tools
  store their config files inside.

## Caveats

- **The vocabulary is the cost.** `jj`'s commands don't map
  1:1 to git commands (`jj squash` ≠ `git commit --amend`,
  `jj new` ≠ `git checkout -b`, bookmarks ≠ branches). Budget
  a week of read-the-docs-while-using-it before muscle memory
  catches up. The [`jj` tutorial](https://jj-vcs.github.io/jj/latest/tutorial/)
  is the right starting point; do it once end-to-end.
- **Working-copy auto-snapshotting can confuse other tools.**
  Editors / formatters that watch the file tree and fire on
  save will see snapshots happen at every `jj` command (since
  every command starts by snapshotting the working copy into
  the current change). Usually fine; rarely interacts badly
  with file-watcher-heavy stacks (some Bazel / Buck setups,
  some IDE indexers).
- **`--colocate` mode has subtle gotchas.** If your teammates
  do operations on the `.git/` directory that `jj` doesn't
  expect (force-push that rewrites a bookmark you have a
  local descendant of, garbage-collection that prunes a
  commit `jj` references), you can end up with `jj` pointing
  at a no-longer-extant commit. `jj git import` reconciles;
  read the [colocation docs](https://jj-vcs.github.io/jj/latest/git-compatibility/)
  before adopting on a busy shared repo.
- **Pre-commit hooks need adapting.** Git's `pre-commit`
  hook fires on `git commit`; `jj` has its own hook system
  that's still maturing (`jj fix` runs configured formatters,
  no first-class pre-commit equivalent yet). For a repo that
  relies heavily on `pre-commit` framework hooks, run them
  from CI rather than expecting `jj` to fire them locally.
- **Pre-1.0 — pin the version.** `jj` is on a steady release
  cadence (v0.x roughly monthly) and the CLI surface still
  shifts between minors. Pin in your team's setup docs
  (`brew install jj@0.40` or `cargo install --locked
  --version 0.40.0 jj-cli`) and review release notes before
  bumping.
