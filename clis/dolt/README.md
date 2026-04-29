# dolt

> **A SQL database with Git-style versioning** —
> a MySQL-wire-compatible relational store where
> every table change is a commit, branches and merges
> work on rows instead of files, and `diff` shows you
> what data moved between two revisions. Pinned to
> **v1.86.6**
> ([LICENSE](https://github.com/dolthub/dolt/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/dolthub/dolt>

## TL;DR

`dolt` is "Git for data" implemented as a real
database, not a filesystem trick. The `dolt` binary
is a single Go executable that exposes two surfaces:
a `git`-shaped CLI (`dolt init`, `dolt add`,
`dolt commit -m`, `dolt branch`, `dolt merge`,
`dolt diff`, `dolt log`, `dolt push`,
`dolt clone <remote>`) operating over tables, and a
MySQL-wire server (`dolt sql-server`) that any
existing client (`mysql`, `mycli`, `harlequin`,
`dbeaver`, `goose`, `dbt`, application drivers) can
connect to as if it were MySQL 8 — but with extra
system functions (`DOLT_COMMIT()`, `DOLT_MERGE()`,
`DOLT_DIFF()`, `DOLT_HISTORY_<table>`) that let SQL
itself drive the version control. Storage is a
content-addressed Merkle DAG over Prolly Trees, so
diffs and merges scale to multi-GB tables and the
on-disk format is canonical (two clones of the same
data have byte-identical chunks). Remotes work like
git's: `dolthub.com` is the public hosted forge,
but you can also push to S3, GCS, Azure Blob, an OCI
registry, or a filesystem path. The whole thing
re-implements MySQL's parser + planner + executor
(via `go-mysql-server` / `vitess`) so it speaks
SQL-92 + most MySQL 8 extensions natively, and ships
TPC-C / sysbench numbers in every release.

## Install

```bash
# Homebrew
brew install dolt

# install script (Linux / macOS)
sudo bash -c 'curl -L https://github.com/dolthub/dolt/releases/latest/download/install.sh | bash'

# Go
go install github.com/dolthub/dolt/go/cmd/dolt@latest

# Windows MSI
# download from https://github.com/dolthub/dolt/releases/latest

# verify
dolt version    # dolt version 1.86.6
```

## Basic usage

```bash
# init a repo, like git init
mkdir crops && cd crops && dolt init

# create a table and stage it
dolt sql -q "CREATE TABLE harvest (id INT PRIMARY KEY, crop VARCHAR(64), kg INT)"
dolt sql -q "INSERT INTO harvest VALUES (1,'wheat',420),(2,'rye',180)"
dolt add harvest
dolt commit -m "initial harvest figures"

# branch, edit, diff, merge — all on rows
dolt branch winter
dolt checkout winter
dolt sql -q "UPDATE harvest SET kg = 0 WHERE crop = 'rye'"
dolt diff main winter
dolt checkout main
dolt merge winter -m "apply winter losses"

# inspect history of a single table / row
dolt sql -q "SELECT * FROM dolt_log"
dolt sql -q "SELECT * FROM dolt_history_harvest WHERE id = 2"
dolt sql -q "SELECT * FROM dolt_diff_harvest WHERE to_commit = HASHOF('HEAD')"

# serve as MySQL — any tool can now connect on :3306
dolt sql-server --host 0.0.0.0 --port 3306 --user root

# clone a public dataset from DoltHub
dolt clone dolthub/us-presidents
cd us-presidents
dolt sql -q "SELECT name, party FROM presidents WHERE term_start > '1900-01-01'"

# remotes
dolt remote add origin https://doltremoteapi.dolthub.com/team/repo
dolt push origin main
```

## When to choose

- **Your data is the artifact, and you need
  human-reviewable history of every change** — audit
  trails, regulatory data, reference tables,
  configuration databases. `git log` on the schema +
  `dolt log` on the rows gives you a single
  reviewable diff for "the world as of last Tuesday."
- **You want CI-style pull-request workflows for
  data** — analysts open branches, propose changes,
  reviewers see row-level diffs in the DoltHub UI or
  `dolt diff`, merge happens with conflict resolution
  on conflicting rows, not file lines. Replaces
  ticket-driven "please update the lookup table"
  rituals.
- **You're shipping a public dataset and want
  reproducible snapshots** — `dolt clone
  org/dataset@<hash>` is byte-identical across
  machines, so a paper, dashboard, or test fixture
  can pin to an exact commit and never drift.
- **You're standing up an OLTP-shaped service that
  needs MySQL compatibility plus time-travel queries**
  — `AS OF '<commit>'` clauses on every `SELECT`,
  per-row history tables, and branch-based blue/green
  schema migrations come for free.

## When NOT to choose

- **You need a drop-in replacement for a high-write
  production MySQL** — Dolt's TPC-C numbers are
  improving release-over-release but still trail
  MySQL 8 by ~2x on write-heavy workloads. Use it for
  data that is read-mostly, audited, or
  human-curated; keep your transactional OLTP on
  MySQL / Postgres / TiDB.
- **You only need file-level versioning of CSVs /
  parquet** — `git` + LFS, `dvc`, or `lakeFS` are
  lighter weight if you don't actually need SQL
  semantics or row-level merges. Dolt earns its
  weight when the questions you ask of the data are
  themselves SQL.
- **You're committed to PostgreSQL dialect** — Dolt
  is MySQL-wire only. The DoltHub team has a
  separate project (`doltgres`) for Postgres
  compatibility, but it's earlier-stage; if you need
  Postgres today, wait or pick a different tool.
- **Your dataset is multi-TB and update-heavy** —
  the Prolly-tree storage shines for diff/merge but
  per-row write amplification is higher than B-tree
  MySQL. Bench before committing.

## Why it fits the zoo

The zoo collects CLIs that take a familiar
developer workflow (edit → commit → review → merge)
and apply it to a new substrate. `dolt` is the
clearest example of that pattern outside source
code: it takes the entire `git` mental model — repo,
branch, remote, commit hash, diff, merge, conflict
resolution — and re-grounds it in SQL tables.
Companion to data-CLI entries already in the zoo
([`harlequin`](../harlequin/), [`usql`](../usql/),
[`mycli`](../mycli/), [`lazysql`](../lazysql/),
[`duckdb`](../duckdb/), [`csvkit`](../csvkit/)) and
to the version-control cluster
([`gitui`](../gitui/), [`lazygit`](../lazygit/),
[`jj`](../jj/), [`gitoxide`](../gitoxide/)) —
Dolt is the bridge that makes "version-controlled
data" a first-class noun rather than a workaround
involving CSV diffs.

## Upstream pointers

- Repo: <https://github.com/dolthub/dolt>
- Release notes: <https://github.com/dolthub/dolt/releases>
- License: [Apache-2.0](https://github.com/dolthub/dolt/blob/main/LICENSE)
- Docs: <https://docs.dolthub.com/>
- Hosted forge: <https://www.dolthub.com/>
- Maintainer: [DoltHub](https://github.com/dolthub)
  (also [`go-mysql-server`](https://github.com/dolthub/go-mysql-server)
  and a Vitess fork).
