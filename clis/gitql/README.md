# gitql

> Snapshot date: 2026-05. Upstream: <https://github.com/filhodanuvem/gitql>

**A SQL-like query language for git repositories.** `gitql` lets you point
SQL — `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`, aggregates — at the four
virtual tables a git repo exposes (`commits`, `refs`, `tags`, `branches`)
and get back rows instead of having to chain `git log --pretty=format:...`
through `awk` / `cut` / `sort -u`. It's a thin Go binary that reads the
on-disk repo via `go-git`, so there's no daemon, no indexing step, and no
external service: `cd` into any repo and start querying.

## Repo + version + license

- Repo: <https://github.com/filhodanuvem/gitql>
- Latest release: **`v2.3.1`** (2023-01-14)
- HEAD on `main`: `9aab2f5`
- License: **MIT** —
  <https://github.com/filhodanuvem/gitql/blob/main/LICENSE>
- License path in repo: `LICENSE` (SPDX: `MIT`)
- Default branch: `main`
- Language: Go (uses `src-d/go-git` for repository traversal, custom lexer/parser for the SQL dialect)

## Install

```bash
# Homebrew
brew install gitql

# Go
go install github.com/filhodanuvem/gitql@latest

# Or grab a release binary
curl -L https://github.com/filhodanuvem/gitql/releases/download/v2.3.1/gitql_2.3.1_$(uname -s)_$(uname -m).tar.gz | tar xz
```

```sql
-- Top 10 committers by raw commit count
gitql "SELECT author, COUNT(*) AS n FROM commits GROUP BY author ORDER BY n DESC LIMIT 10"

-- Every tag that points to a commit by a specific author in the last year
gitql "SELECT name, hash FROM tags WHERE author = 'Jane Doe' AND date > '2025-05-01'"

-- Branches that haven't moved in 6 months (stale-branch hunt)
gitql "SELECT name, hash FROM branches WHERE date < '2025-11-01' ORDER BY date"
```

## Niche

The "**SQL over git history**" slot. Most people answer the same questions
(`who touched this file most?`, `what tags shipped between two dates?`,
`which branches are stale?`) by stringing together `git log`,
`git for-each-ref`, `awk`, `sort`, and `uniq -c`. Once the query gets
non-trivial — joins between commits and refs, an aggregate on top of a
filter — that pipeline becomes write-only. `gitql` replaces the pipeline
with a 1-line SQL statement and a familiar mental model.

## Why it matters

- **Familiar query surface, no schema work** — the four built-in tables
  (`commits`, `refs`, `tags`, `branches`) are auto-discovered from the
  `.git/` directory; you don't write a schema, run a sync, or stand up a
  database. The query plan is a streaming traversal of the object graph
  via `go-git`, so a 100k-commit repo answers most queries in seconds
  without loading the whole graph into RAM.
- **Output formats for piping** — supports JSON and CSV output (`-f json`
  / `-f csv`) on top of the default human table, so `gitql` slots cleanly
  into shell pipelines: dump committers as JSON, pipe into `jq`, post to
  a dashboard. The interactive REPL (`gitql` with no args, in a repo) is
  there for exploration.
- **Comparable CLIs** —
  [`onefetch`](../onefetch/) gives you a one-shot "repo summary card" but
  isn't queryable. [`git-sizer`](../git-sizer/) measures repo health, not
  history. [`scc`](../scc/) and [`tokei`](../tokei/) count code, not
  commits. [`steampipe`](../steampipe/) has a GitHub plugin that exposes
  the *GitHub API* as SQL (issues, PRs, workflow runs), which is the
  remote-flavored cousin of what `gitql` does locally over the on-disk
  git database. Pick `gitql` when you want offline, repo-local SQL with
  no API tokens and no network.
- **MIT, single static Go binary** — ~10 MB, no runtime dependencies,
  works on any platform Go cross-compiles to.
