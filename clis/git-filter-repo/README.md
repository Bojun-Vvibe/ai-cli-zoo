# git-filter-repo

> **Fast, scriptable git history rewriter — the modern
> replacement for `git filter-branch` and BFG, recommended by
> the git project itself.** Pinned to **v2.47.0**
> ([COPYING.mit](https://github.com/newren/git-filter-repo/blob/main/COPYING.mit),
> MIT; also dual-available under GPL-2.0 via
> [COPYING.gpl](https://github.com/newren/git-filter-repo/blob/main/COPYING.gpl)).

Source: <https://github.com/newren/git-filter-repo>

## TL;DR

`git-filter-repo` is a single Python script that rewrites the
history of a git repository — strip a leaked secret, extract a
subdirectory into its own repo, drop a giant binary that's
bloating clones, rename an author across every commit, splice
two repos together while keeping history. It is the tool
`git filter-branch`'s own man page now points users at: orders
of magnitude faster (uses `git fast-export`/`fast-import`
under the hood instead of re-running shell per commit), with
saner defaults that don't silently corrupt your history. It
also subsumes most of what `BFG Repo-Cleaner` did, in a single
binary that doesn't require the JVM.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install git-filter-repo

# Debian / Ubuntu (24.04+)
sudo apt install git-filter-repo

# pip (cross-platform; gets you the latest)
pipx install git-filter-repo

# Direct: it's a single Python file, drop it on PATH
curl -L https://raw.githubusercontent.com/newren/git-filter-repo/v2.47.0/git-filter-repo \
  -o /usr/local/bin/git-filter-repo && chmod +x /usr/local/bin/git-filter-repo

# verify
git filter-repo --version
```

## Examples

```bash
# Strip a leaked AWS key from every commit on every branch
git filter-repo --replace-text <(echo 'AKIA****************==>REDACTED')

# Extract one subdirectory into a fresh repo, preserving its history
git filter-repo --subdirectory-filter packages/parser

# Drop every blob bigger than 10 MB (the BFG use case)
git filter-repo --strip-blobs-bigger-than 10M

# Rewrite author/committer across all history
git filter-repo --mailmap <(echo 'New Name <new@example.com> <old@example.com>')

# Move every file under src/ to a new top-level layout
git filter-repo --path-rename src/:lib/
```

## When to choose it

Reach for it the moment you need to rewrite *committed*
history — secrets that already landed on `main`, a
subdirectory that should be its own repo, a 2 GB pack file
from a vendored binary you no longer want, an author email
that needs to change everywhere. It is **destructive by
design** (the resulting repo has new SHAs) so it is for the
"we will force-push and tell collaborators to re-clone" path,
not the "I'll just amend the last commit" path.

Skip it for in-flight changes — `git rebase -i`, `git commit
--amend`, and `git revert` cover the not-yet-pushed cases
without breaking other people's clones. Also skip it for very
small one-shot edits where `git filter-branch` muscle memory
would be faster to type; `filter-repo` is meant for repeatable
or large-scale rewrites where you want speed and safety rails.

## Vs adjacent tools

- **Vs `git filter-branch`:** built-in to git, but slow
  (shells out per commit), full of subtle pitfalls, and
  upstream-deprecated — its own man page recommends
  `git-filter-repo`. Use `filter-repo` instead in 2025.
- **Vs `BFG Repo-Cleaner`:** Java tool focused on the "delete
  big files / replace strings" cases. `filter-repo` covers
  those plus the harder structural rewrites (path renames,
  subdirectory extraction, mailmap) and has no JVM dep.
- **Vs `git rebase -i` / `git commit --amend`:** those edit
  *not yet shared* commits. `filter-repo` rewrites *published*
  history wholesale and forces a re-clone. Different problem.
- **Vs [`git-sizer`](../git-sizer/):** `git-sizer` *measures*
  what is bloating your repo (big blobs, deep history, fat
  trees). `filter-repo` is what you run after `git-sizer`
  tells you what to remove.
- **Vs [`gitleaks`](../gitleaks/) / [`trufflehog`](../trufflehog/):**
  those *find* leaked secrets in history. `filter-repo` is
  the surgery you run after the scan tells you which strings
  to redact.

## Caveats

- **It rewrites SHAs.** Every commit downstream of the edit
  gets a new ID. Coordinate force-pushes with everyone who
  has clones, and update any external pointers (CI configs,
  release tags, "fixed in commit abc123" tickets).
- **Refuses to run on a non-fresh clone by default.** This is
  a feature: a fresh clone is the safest place to rewrite, and
  it stops people from accidentally running it on a working
  repo with uncommitted changes. Use `--force` if you really
  mean it.
- **Tags and refs need re-pushing.** After a rewrite, push
  with `git push --force --tags` (or `--mirror` to a fresh
  remote) so tags follow the new commit graph.
- **Secrets that already shipped are still leaked.** Rewriting
  history removes them from future clones, but anyone who had
  the old history (CI caches, forks, mirrors) still has the
  secret. Rotate the credential first; rewrite history second.
