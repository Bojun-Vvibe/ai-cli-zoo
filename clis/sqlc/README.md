# sqlc

> **Type-safe Go (and Kotlin / Python / TypeScript) code generated
> from plain SQL** — you write `SELECT`, `INSERT`, `UPDATE`,
> `DELETE` queries in `.sql` files, point `sqlc generate` at them,
> and get back fully-typed query methods, request/response structs,
> and a database interface — no ORM, no reflection, no string
> concatenation in your service code, just compile-time-checked
> SQL. Pinned to **v1.31.1**
> ([LICENSE](https://github.com/sqlc-dev/sqlc/blob/main/LICENSE),
> MIT).

Source: <https://github.com/sqlc-dev/sqlc>

## TL;DR

`sqlc` inverts the ORM contract: instead of declaring entities in
the host language and asking a runtime to translate method calls
into SQL, you write the SQL by hand and ask a build-time tool
to translate it into typed host-language code. The workflow is:
write a schema (`schema.sql` — the same `CREATE TABLE` statements
you would feed to `psql` or migrate with `dbmate`/`goose`),
write your queries (`query.sql` — annotated with `-- name:
GetUser :one` / `:many` / `:exec` directives), declare a
`sqlc.yaml` config pointing at both, and run `sqlc generate`.
Out comes a `db.go` package with `type Queries struct`,
`func (q *Queries) GetUser(ctx, id) (User, error)`, and a
`Querier` interface you can mock in tests. Engines: PostgreSQL
(first class, full parser), MySQL, SQLite. Languages: Go (stable),
Kotlin / Python / TypeScript (via plugins). The generated code
has zero runtime dependency on `sqlc` itself — ship the binary in
CI, ship the generated `.go` in the repo.

## Install

```bash
# Homebrew
brew install sqlc

# Go install
go install github.com/sqlc-dev/sqlc/cmd/sqlc@v1.31.1

# Static binary from GitHub Releases (linux / macOS / windows, amd64 / arm64)
curl -fsSL -o /tmp/sqlc.tar.gz \
  https://github.com/sqlc-dev/sqlc/releases/download/v1.31.1/sqlc_1.31.1_$(uname -s | tr A-Z a-z)_arm64.tar.gz
tar -xzf /tmp/sqlc.tar.gz -C /usr/local/bin sqlc

# Docker (great for CI — no toolchain pin)
docker run --rm -v "$(pwd)":/src -w /src sqlc/sqlc:1.31.1 generate

# verify
sqlc version    # v1.31.1
```

## One Concrete Example

```yaml
# sqlc.yaml
version: "2"
sql:
  - engine: postgresql
    schema: db/schema.sql
    queries: db/query.sql
    gen:
      go:
        package: db
        out: internal/db
        sql_package: pgx/v5
        emit_interface: true
        emit_json_tags: true
```

```sql
-- db/schema.sql
create table authors (
  id   bigserial primary key,
  name text not null,
  bio  text
);

-- db/query.sql
-- name: GetAuthor :one
select * from authors where id = $1;

-- name: ListAuthors :many
select * from authors order by name;

-- name: CreateAuthor :one
insert into authors (name, bio) values ($1, $2) returning *;
```

```bash
sqlc generate
# writes internal/db/{db.go,models.go,query.sql.go}
```

```go
// usage in service code
import "your/module/internal/db"

q := db.New(pool) // pool is *pgxpool.Pool
a, err := q.CreateAuthor(ctx, db.CreateAuthorParams{
    Name: "Borges",
    Bio:  pgtype.Text{String: "Argentine", Valid: true},
})
```

If you misname a column in `query.sql`, **`sqlc generate`** fails
with a parser error citing the SQL line — not a 3 a.m. runtime
panic. If you change the schema and forget to regenerate, the
build fails because `db.CreateAuthorParams` no longer matches.

## Niche It Fills

**The "I want types, but I refuse to give up handwritten SQL"
escape hatch.** GORM / Ent / Prisma / SQLAlchemy hide the SQL
behind a query-builder DSL — fast for CRUD, painful when you
need a window function, a CTE, or a recursive query and the DSL
doesn't speak it. Raw `database/sql` keeps the SQL but loses
types — every `rows.Scan` is a manual exercise in column-order
discipline. `sqlc` is the third corner of that triangle:
**SQL you author, types you don't**.

## Vs Already Cataloged

- **Vs [`dbmate`](../dbmate/) / [`goose`](../goose/) /
  [`sqlx-cli`](../sqlx-cli/):** those manage *schema migrations*
  (apply `CREATE TABLE` over time). `sqlc` reads the *current*
  schema and generates *query code*. They compose: `dbmate up`
  to apply migrations, `sqlc generate` to regenerate the typed
  query layer against the new schema, then commit both.
- **Vs [`atlas`](../atlas/):** `atlas` is schema-as-code +
  drift detection (declare desired schema, plan & apply diff).
  `sqlc` is query-code-from-schema. Pair them: `atlas` owns
  what the schema *is*, `sqlc` owns what your code *does* with
  that schema.
- **Vs [`sqlfluff`](../sqlfluff/):** `sqlfluff` lints the SQL
  text. `sqlc` parses it, type-checks it against the schema,
  and emits Go. Run `sqlfluff` in pre-commit, `sqlc generate
  --check-only` in CI.
- **Vs [`harlequin`](../harlequin/) / [`usql`](../usql/):**
  interactive REPLs for ad-hoc query exploration. Use them
  to *discover* the query you want, paste it into
  `db/query.sql`, run `sqlc generate`.

## Caveats

- Queries must be statically analyzable — `sqlc` parses them at
  build time, so dynamic `WHERE` clauses ("filter by N optional
  fields") need either multiple named queries (`ListUsersByName`
  / `ListUsersByEmail`) or a `sqlc.arg()` + `coalesce` trick.
  If you live and die on dynamic predicates, a query-builder
  (Squirrel, goqu) is a better fit — possibly *alongside* sqlc
  for the static 80%.
- The PostgreSQL engine is the most mature — it ships its own
  Postgres parser fork. MySQL and SQLite work, but edge-case
  syntax (e.g. some MySQL JSON ops, SQLite full-text virtual
  tables) may need an `-- name: X :exec` escape hatch with
  `sqlc.arg`-only typing.
- Generated code is checked into the repo (recommended) — your
  diff includes both `query.sql` and `query.sql.go`. Reviewers
  should treat generated files as machine-output (no manual
  edits) and check the *source* SQL.
- Driver coupling matters — pick `pgx/v5` (default for new
  Postgres projects), `database/sql` (portable), or `pgx/v4`
  (legacy) at config time. Switching later means regenerating
  every `.sql.go` and updating call sites where types differ
  (`pgtype.Text` vs `sql.NullString`).
- Plugin languages (Kotlin / Python / TypeScript) are
  implemented as out-of-process WASM / native plugins; they
  lag Go in features and stability. For non-Go targets, audit
  the plugin's status before committing.
