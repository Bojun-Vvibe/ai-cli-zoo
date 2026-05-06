# neonctl

> **neonctl** — neondatabase/neonctl's official command-line client
> for Neon, the serverless Postgres platform with branching: create
> projects, branches, endpoints, databases, roles, and run SQL from
> the terminal without ever opening the Neon console. Pinned to
> **v2.22.0**, Apache-2.0 — license file:
> [LICENSE](https://github.com/neondatabase/neonctl/blob/main/LICENSE).

Source: <https://github.com/neondatabase/neonctl>

## TL;DR

Neon's pitch is **branchable Postgres**: every database is
copy-on-write at the storage layer (their own page-server / safekeeper
fork) so `neonctl branches create --name preview-pr-42 --parent main`
gives you a full Postgres database that started as an instant
zero-copy snapshot of `main`, costs nothing until you write to it,
and can be torn down at PR close. `neonctl` is the surface that makes
that workflow scriptable from CI:

- `neonctl projects` / `branches` / `endpoints` / `databases` /
  `roles` / `operations` — full CRUD over the Neon control plane.
- `neonctl connection-string <branch>` — emits the right
  `postgres://...?sslmode=require` URL for a branch+role+database
  triple, so a CI step is `export DATABASE_URL=$(neonctl
  connection-string preview-pr-42)`.
- `neonctl ip-allow` — manage the per-project IP allowlist from the
  same CLI that creates the branches.
- `neonctl auth` does an interactive OAuth dance once and then caches
  a refresh token in `~/.config/neonctl/credentials.json`; CI uses
  `--api-key $NEON_API_KEY` instead.
- `--output json` on every verb gives you the structured response
  for piping into `jq` / further automation.

The killer use is **per-PR preview databases**: the GitHub Action
calls `neonctl branches create` on PR open, sets `DATABASE_URL` to
the branch's connection string, runs migrations + tests + a deploy
preview against real production-shaped data, and `neonctl branches
delete` on PR close. Each branch is a few cents of metadata until
something writes to it.

## Install

```bash
# npm — the canonical install path (Node 18+)
npm install -g neonctl

# npx for one-shot use
npx neonctl projects list

# Homebrew
brew install neonctl

# Pre-built binaries (Linux x64 / arm64, macOS, Windows)
# https://github.com/neondatabase/neonctl/releases/tag/v2.22.0
```

## Example commands

```bash
# Auth once interactively
neonctl auth

# List projects
neonctl projects list

# Create a preview branch off main
neonctl branches create --name preview-pr-42 --parent main \
  --project-id ancient-leaf-12345

# Get the connection string for the new branch
export DATABASE_URL=$(neonctl connection-string preview-pr-42 \
  --project-id ancient-leaf-12345)

# Run migrations against it
psql "$DATABASE_URL" -f migrations.sql

# Tear it down
neonctl branches delete preview-pr-42 --project-id ancient-leaf-12345

# In CI, prefer an API key over interactive auth
neonctl --api-key "$NEON_API_KEY" branches list --output json | jq .
```

## Niche it occupies

**Per-branch serverless Postgres CLI** — different niche from the
rest of the catalog. Closest neighbours:

- [`turso-cli`](../turso-cli/) — Turso's CLI for libSQL/SQLite-edge
  databases. Same per-branch / per-PR shape, different engine
  (SQLite-on-the-edge vs. Postgres-on-managed-storage). Pick neonctl
  when you need real Postgres semantics (extensions, `pg_trgm`,
  PostGIS, full-text search via tsvector); pick turso for SQLite +
  edge-replicated reads.
- [`flyctl`](../flyctl/) / [`wrangler`](../wrangler/) /
  [`railway`](https://docs.railway.com/reference/cli-api) — platform
  CLIs that *can* provision a Postgres instance as one resource
  among many. Pick neonctl when Neon-the-database is the centre of
  the workflow and branching is the primary verb (the platform CLIs
  do not expose branch-create at all).
- [`dbmate`](../dbmate/) / [`golang-migrate`](../golang-migrate/) /
  [`atlas`](../atlas/) / [`pgroll`](../pgroll/) — schema-migration
  tools. Orthogonal: those *apply* migrations to a database;
  `neonctl` *creates* the database (and `neonctl branches create
  --parent main` is itself a kind of zero-cost "snapshot before
  migration" for safety).
- [`pgcli`](../pgcli/) / [`psql`](https://www.postgresql.org/docs/current/app-psql.html)
  — interactive Postgres clients. Compose: `psql "$(neonctl
  connection-string main)"` is the canonical "open a shell on the
  current branch" pattern.

Pairs cleanly with [`pgbackrest`](../pgbackrest/) /
[`litestream`](../litestream/) (off-Neon backup paths if you want
your own copy outside the platform), with [`dbmate`](../dbmate/) /
[`atlas`](../atlas/) (per-branch migration runners in CI), and with
[`gh`](../gh/) (the GitHub Action that wires `gh pr` events to
`neonctl branches create` / `delete` is a few lines of YAML).

Caveats: Neon-only — this CLI does not work against vanilla
Postgres, RDS, or Aurora; the value proposition *is* the
branching/copy-on-write storage layer, which is proprietary to Neon;
free-tier projects have a branch limit (10 at time of writing) that
busy monorepos hit quickly; auto-suspend (the "scale to zero"
behaviour that makes idle branches free) adds cold-start latency
(~500 ms) on the first query after idle — fine for CI, surprising
for a human staring at a pgcli prompt.

## Citation

- Repo: <https://github.com/neondatabase/neonctl>
- Latest release: **v2.22.0**
- License: **Apache-2.0**
- License file: [LICENSE](https://github.com/neondatabase/neonctl/blob/main/LICENSE)
