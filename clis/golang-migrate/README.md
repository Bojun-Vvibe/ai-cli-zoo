# golang-migrate

> **A database-agnostic schema migration CLI driven by
> ordered, paired `up`/`down` SQL files** — `migrate -path
> ./migrations -database $DSN up` applies every pending
> versioned migration inside a transaction (when the engine
> supports it) and records the new version in a tracking table;
> a single static Go binary speaks PostgreSQL, MySQL, SQLite,
> CockroachDB, ClickHouse, MongoDB, Redshift, Spanner, and a
> dozen more.
> Pinned to **v4.19.1**
> ([LICENSE](https://github.com/golang-migrate/migrate/blob/master/LICENSE),
> MIT).

Source: <https://github.com/golang-migrate/migrate>

## TL;DR

`migrate` (the CLI from `golang-migrate/migrate`) is the
schema-evolution tool that wins on portability and
transparency. Migrations are plain `.sql` files numbered by
timestamp or sequence, paired as `000123_add_users.up.sql` and
`000123_add_users.down.sql`. There is no DSL, no ORM coupling,
no embedded YAML — what you write is what runs. The CLI tracks
applied versions in a `schema_migrations` table per database
engine, supports forced version reset for emergencies (`force`),
partial application (`up N` / `down N`), and "go to specific
version" (`goto`). It's the de-facto migration tool in the Go
ecosystem and is widely used from non-Go projects too because
the CLI doesn't care what language your app is written in.

## Install

```bash
# Homebrew (macOS / Linux)
brew install golang-migrate

# Pre-built release tarball
curl -LO "https://github.com/golang-migrate/migrate/releases/download/v4.19.1/migrate.darwin-arm64.tar.gz"
tar -xzf migrate.darwin-arm64.tar.gz
sudo mv migrate /usr/local/bin/

# Linux package managers
# Debian/Ubuntu: dpkg -i migrate.linux-amd64.deb
# Arch (AUR):    yay -S migrate

# Go install with build tags for the drivers you need
go install -tags 'postgres mysql sqlite3' \
  github.com/golang-migrate/migrate/v4/cmd/migrate@v4.19.1

# Docker
docker run --rm -v "$PWD/migrations:/migrations" \
  migrate/migrate:v4.19.1 \
  -path=/migrations \
  -database "postgres://..." up

# verify
migrate -version    # 4.19.1
```

The Homebrew bottle ships with the common drivers compiled in.
For Go-install builds, the `-tags` flag selects which database
drivers are linked — important because pulling in every driver
balloons the binary unnecessarily.

## License

MIT — see
[LICENSE](https://github.com/golang-migrate/migrate/blob/master/LICENSE).
Permissive; vendoring the CLI in a release artifact, embedding
in a deploy image, or shipping it as part of a paid product is
all fine.

## One Concrete Example

```bash
# 1. scaffold a new migration pair
migrate create -ext sql -dir db/migrations -seq add_users_table
# creates: db/migrations/000001_add_users_table.up.sql
#          db/migrations/000001_add_users_table.down.sql

# 2. write the SQL
cat > db/migrations/000001_add_users_table.up.sql <<'SQL'
CREATE TABLE users (
  id          BIGSERIAL PRIMARY KEY,
  email       TEXT NOT NULL UNIQUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX users_email_idx ON users (email);
SQL

cat > db/migrations/000001_add_users_table.down.sql <<'SQL'
DROP INDEX IF EXISTS users_email_idx;
DROP TABLE IF EXISTS users;
SQL

# 3. apply all pending migrations
export PG_DSN="postgres://app:secret@localhost:5432/app?sslmode=disable"
migrate -path db/migrations -database "$PG_DSN" up

# 4. apply only the next 1 migration
migrate -path db/migrations -database "$PG_DSN" up 1

# 5. roll back the most recent migration
migrate -path db/migrations -database "$PG_DSN" down 1

# 6. jump to a specific version
migrate -path db/migrations -database "$PG_DSN" goto 5

# 7. inspect current state
migrate -path db/migrations -database "$PG_DSN" version
# 7

# 8. recover from a dirty state (a migration partially applied
#    then crashed); manually verify schema first, then:
migrate -path db/migrations -database "$PG_DSN" force 7

# 9. CI / production: run migrations as a Job before app deploy
docker run --rm \
  -v "$PWD/db/migrations:/m" \
  --network host \
  migrate/migrate:v4.19.1 \
  -path /m -database "$PG_DSN" up
```

## Niche It Fills

**Plain-SQL, language-agnostic schema migrations with strong
multi-engine support.** When your team is polyglot (Go service +
Python worker + Node API all hitting the same Postgres) you do
not want a Rails-style migration DSL tied to one ORM. `migrate`
gives you raw `.sql` files that any engineer can read and any
DBA can sanity-check, plus a single binary that runs identically
in dev, CI, and prod. The same tool can also handle MySQL,
SQLite, CockroachDB, ClickHouse, and Spanner, so polyrepo /
poly-engine shops standardize on one workflow.

## Why use it

Three things `migrate` does that explain its survival next to
ORM-bundled migration tools (Django, Rails AR, Prisma, Alembic,
Knex):

1. **Plain `.sql` files, no DSL, no codegen.** Every migration
   is a SQL script you can `psql -f` by hand if the tool ever
   broke. There is no "migration definition file" that gets
   compiled to SQL at runtime — you write the actual statements
   the database will execute. This makes code review trivial
   for DBAs and audit trails clean.
2. **Many sources, many databases, one binary.** Migration
   files can come from the local filesystem, a GitHub repo, an
   S3 bucket, a Go embed.FS, or a `go-bindata` blob — useful
   for shipping migrations inside a single deploy artifact.
   Targets include PostgreSQL, MySQL, MariaDB, SQLite,
   CockroachDB, ClickHouse, MongoDB, Redshift, Spanner,
   YugabyteDB, Cassandra, Neo4j, Firebird, and more. One CLI,
   one mental model.
3. **Honest version tracking with explicit recovery.** The
   `schema_migrations` table stores `(version, dirty)`. If a
   migration crashes mid-apply, the version is marked `dirty`
   and the CLI refuses further operations until you investigate
   and `force` it to a known-good version. This is more honest
   than tools that silently retry or that store version state in
   the application binary.

For an LLM-CLI workflow that generates schema changes, having
the agent emit `*.up.sql` / `*.down.sql` pairs and then call
`migrate up` is much cleaner than asking it to drive an ORM —
the SQL is the artifact, reviewable in PR, replayable on any
environment.

## Vs Already Cataloged

- **Vs [`atlas`](../atlas/):** Atlas (from ariga.io) is the
  modern competitor — declarative HCL/SQL schemas, drift
  detection, schema linting, and managed Atlas Cloud features.
  Atlas is more powerful when you want "describe the desired
  state, let the tool diff and plan"; `migrate` is more
  appropriate when you want "I write the exact SQL that runs,
  in order, every time". Pick Atlas for declarative
  schema-as-state workflows; pick `migrate` for imperative
  versioned-script workflows and the broadest engine coverage.
- **Vs ORM-bundled migrations (Django/Rails/Prisma/Alembic):**
  ORM migrations couple schema state to your application
  language and framework version. `migrate` decouples them, so
  a polyglot stack or a future framework rewrite doesn't
  invalidate your migration history. Pick ORM tools for
  single-language monorepos where the ORM is already the source
  of truth; pick `migrate` for shared databases or when SQL
  fidelity matters.
- **Vs `flyway` (not cataloged):** Flyway is the JVM-world
  equivalent — also plain SQL, also versioned, also widely
  deployed. Tradeoffs: Flyway requires a JVM (heavier image,
  slower startup), has a paid Teams edition with extra
  features, and dominates Java shops. `migrate` is a 20 MB Go
  static binary, fully open source, and dominates Go/polyglot
  shops. Functionally near-equivalent for the common case;
  pick by ecosystem.

## Caveats

- **Driver build tags matter for `go install`.** The default
  `go install` build only includes a subset of drivers. If you
  see `unknown driver clickhouse`, rebuild with
  `-tags 'clickhouse'`. The Homebrew bottle and Docker image
  include the common set; the `release` tarballs include all
  drivers.
- **`down` migrations are not magic.** A `DROP TABLE users`
  loses data. The CLI runs whatever SQL is in the `.down.sql`
  file — there is no automatic rollback safety net. Treat
  `down` as a tool for development reversibility, not as a
  production safety mechanism. Use database backups for that.
- **Dirty state requires manual intervention.** When a
  migration fails mid-apply (especially on engines without DDL
  transactions like MySQL), `migrate` marks the version
  `dirty` and stops. The fix — inspect schema, repair by hand
  if needed, then `force <last_good_version>` — is correct but
  surprises first-time users. Make sure your runbook has a
  "dirty migration" section.
- **No automatic dependency ordering between files.** Order is
  strictly numeric: `000123_*` runs before `000124_*`, period.
  Concurrent feature branches that both create migration
  numbered `000125_*` will collide on merge. Use timestamp-
  based naming (`migrate create -seq` is sequence-based;
  `-format` lets you switch to timestamps) on busy teams.
- **DSN parsing is per-driver-quirky.** Postgres
  `?sslmode=disable`, MySQL `?multiStatements=true`,
  ClickHouse `?x-multi-statement=true`, etc. Read the per-
  driver README in the upstream repo before debugging
  connection issues — most "doesn't work" reports are missing
  query-string flags.
