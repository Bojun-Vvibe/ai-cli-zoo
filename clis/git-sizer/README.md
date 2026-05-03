# git-sizer

> **Repo-health auditor that walks the entire git object graph in
> one pass and surfaces the specific objects, paths, and history
> shapes that make a repository slow, expensive to clone, or
> hostile to tooling — outputting a per-dimension scorecard
> (largest blob, deepest tree, longest path, most parents, total
> object count) with a 0–30 severity per axis so you know which
> single fix unlocks the next 10× of clone speed.** Pinned to
> **v1.5.0**, MIT
> ([LICENSE.md](https://github.com/github/git-sizer/blob/master/LICENSE.md)).

- **Repo:** https://github.com/github/git-sizer
- **Latest version:** v1.5.0
- **License:** MIT (`LICENSE.md` at repo root)
- **Category:** `git-tooling` / `repo-health` / `defensive-audit`
- **Language:** Go

## What it does

`git-sizer` answers the question "why is this repo painful?" by
walking every object in `.git/objects` exactly once and computing
a structural fingerprint along ten dimensions: total commit count,
total tree count, total blob count, total tag count, max blob
size, max tree entries, max path depth, max path length,
max-parent commit (i.e. octopus merges), and total uncompressed
size. Each dimension is mapped to a 0–30 severity score using
empirical thresholds derived from the corpus of real-world
problem repos GitHub's infrastructure team has had to triage,
and the worst-scoring axis tells you precisely which property is
costing you clone / fetch / index time. The classic findings:
"largest blob = 850 MB" means somebody committed a binary —
rewrite history with `git-filter-repo`; "max tree entries =
180,000" means a single directory has too many siblings — split
it; "total object count = 8 M" means history is too long — try
shallow clones or partial-clone filters; "max parents = 47" means
somebody made a 47-way octopus merge that no UI can render.
Output is human-readable by default and JSON with `--json` for
ingest into a dashboard / CI gate.

## Install

```bash
# macOS
brew install git-sizer

# Cross-platform (Go binary, no runtime)
go install github.com/github/git-sizer@latest

# Or grab a prebuilt release tarball
gh release download v1.5.0 --repo github/git-sizer
```

## Examples

```bash
# Run against the current repo with verbose output
git-sizer --verbose

# JSON output for a CI gate / dashboard
git-sizer --json --json-version=2 > repo-health.json

# Run against a remote repo without a full clone
git clone --bare --filter=blob:none https://github.com/torvalds/linux.git
git-sizer -C linux.git --verbose
```

## Why it matters in an AI-native workflow

Coding agents are I/O-bound on git operations far more than humans
notice — `claude-code`, `aider`, `codex`, `cline`, and friends
re-clone or re-index repos on every fresh sandbox / container,
and a 4 GB monorepo with 8 M objects can burn 90 s of an agent's
wall-clock budget before the first prompt token is generated.
`git-sizer` is the diagnostic that tells you which single
property to fix to halve that. Run it once per repo as part of an
"is this repo agent-ready?" health check; gate it in CI with
`--json` parsed against per-axis thresholds (e.g. fail if any
score goes above 25) so a future "oops I committed
node_modules.tar.gz" PR is rejected before it lands and
permanently inflates clone time for every agent run. Pairs with
[`gitleaks`](../gitleaks/) and [`trufflehog`](../trufflehog/)
(secret scanning — orthogonal axis of repo health), [`scc`](../scc/)
(line-of-code accounting — what is in the repo), and
[`onefetch`](../onefetch/) (cosmetic repo summary — git-sizer is
the operational equivalent). Defensive-audit only — it walks the
object graph, never executes hooks or remote refs.
