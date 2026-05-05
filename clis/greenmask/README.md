# greenmask

> **A Postgres-aware logical-dump tool that anonymizes
> production data on the way out, so the dev / staging / CI
> dump is realistic but not real** — wraps `pg_dump` /
> `pg_restore`, reads a YAML config that names the columns to
> transform and the transformer to apply (fake names, fake
> emails, hash, mask, regenerate UUIDs, replace with statistical
> noise, generate synthetic rows), and emits a standard
> `pg_dump`-format archive that restores into any Postgres with
> `pg_restore` — no app changes, no schema changes, no
> destination plugin. Pinned to **v0.2.19** (SPDX:
> `Apache-2.0`,
> [LICENSE](https://github.com/GreenmaskIO/greenmask/blob/main/LICENSE)).

Source: <https://github.com/GreenmaskIO/greenmask>

## TL;DR

`greenmask` solves a problem every team eventually hits: dev /
staging / CI databases need to look like prod (real cardinality,
real edge-case shapes, real query plans), but cannot **be**
prod (PII, payment info, third-party data subject to GDPR /
HIPAA / SOC2 / contract clauses). The choices used to be (a)
hand-roll a `pg_dump | sed` pipeline that always misses one
column, (b) buy a commercial tool, or (c) use a fake-data
generator that produces a database whose query plans bear no
resemblance to prod. greenmask sits in the gap: real
`pg_dump`-format archive, transformations declared per-column
in YAML, dozens of built-in transformers, deterministic seeded
output for reproducibility, and the resulting archive restores
with stock `pg_restore`.

## Install

```bash
# Homebrew
brew install greenmask

# From a release binary
curl -L -o greenmask.tar.gz \
  https://github.com/GreenmaskIO/greenmask/releases/download/v0.2.19/greenmask-linux-amd64.tar.gz
tar xzf greenmask.tar.gz && sudo mv greenmask /usr/local/bin/

# Docker
docker run --rm -it \
  -v $(pwd)/config.yml:/app/config.yml \
  greenmask/greenmask:v0.2.19 dump --config /app/config.yml

# Verify
greenmask --version
```

## Usage

```bash
# 1. Minimal config: anonymize the users.email column on dump
cat > config.yml <<'EOF'
common:
  pg_bin_path: /usr/local/opt/postgresql@16/bin
  tmp_dir: /tmp

storage:
  type: directory
  directory:
    path: ./dump

dump:
  pg_dump_options:
    dbname: "host=prod-db user=readonly dbname=app sslmode=require"

  transformation:
    - schema: public
      name: users
      transformers:
        - name: ReplaceWithFaker
          params:
            column: email
            generator: email
        - name: NoiseFloat
          params:
            column: lifetime_value_usd
            ratio: 0.1
        - name: Hash
          params:
            column: external_id
            function: sha256
EOF

# 2. Dump (greenmask runs pg_dump under the hood, applies the
#    transformers as the COPY stream flows through it, writes
#    archives to ./dump/<timestamp>/)
greenmask --config config.yml dump

# 3. List local dumps
greenmask --config config.yml list-dumps

# 4. Restore into the local dev DB
greenmask --config config.yml restore latest \
  --pg-restore-options "dbname=app_dev"

# 5. Show the schema-level diff between two stored dumps
greenmask --config config.yml show-schema --dump-id <id>

# 6. Validate that no row contains a literal pre-anonymization
#    value (catches "I forgot to declare a transformer" bugs)
greenmask --config config.yml validate \
  --rows-limit 1000 \
  --diff
```

## Why it matters

The "realistic but not real" dev database is a chronically
under-served need. **Faker-only solutions** (Postgres-Faker,
fakedata-pg, generate fresh rows in a clean schema) produce
databases whose distributions, cardinalities, and join-key
overlaps look nothing like prod — every query plan is wrong,
every "this is slow in prod but fast locally" bug is
unreproducible. **Hand-rolled `pg_dump | sed`** misses columns,
mishandles JSON / `ARRAY[]` / `tsvector`, and produces a dump
the next engineer cannot trust. **Commercial tools** (Tonic,
Delphix, Solix) solve it but at five-figure annual costs and
with their own restore plumbing. greenmask sits in the gap —
real prod cardinality, declarative per-column transformations,
deterministic seeds for reproducibility, output is a stock
`pg_dump` archive — and it is open-source, single-binary, runs
in CI.

## Niche It Fills

**Declarative Postgres anonymization on the dump path, output
is a standard `pg_dump` archive.** The space splits four ways:
fake-data generators (`Postgres-Faker`, `pgbench`-with-rows —
fast, but the data looks nothing like prod), hand-rolled
`pg_dump | sed` pipelines (fragile, miss columns), commercial
masking platforms (Tonic / Delphix / Solix — works, but priced
for enterprises), and greenmask (open-source, declarative,
restores via stock `pg_restore`). It is the canonical pick when
the requirement is "a developer-machine restore of last night's
prod DB without violating the data processing addendum."

## Vs Already Cataloged

- **Vs [`pgbackrest`](../pgbackrest/) / [`wal-g`](../walg/) /
  [`barman`](https://www.pgbarman.org/):** these are
  *backup/restore* tools — they preserve the bytes faithfully
  for disaster recovery. greenmask is the opposite: it
  deliberately changes bytes so the restore is *not* faithful
  in the PII columns. Pair them — pgbackrest / wal-g for prod
  backup, greenmask for dev / staging refreshes from the same
  source.
- **Vs [`gh-ost`](../gh-ost/) / [`pgroll`](../pgroll/):** those
  are online schema-migration tools (rewrite tables on a live
  primary). greenmask does not touch the source — it only
  reshapes the dump on the way out. Orthogonal: gh-ost / pgroll
  for prod schema changes, greenmask for prod-to-dev data
  refresh.
- **Vs [`pgcli`](../pgcli/) / [`mycli`](../mycli/):** those are
  interactive shells — useful for *inspecting* the masked
  result, not for producing it. Pipeline: greenmask produces
  the masked dump, `pg_restore` loads it, pgcli pokes at the
  loaded DB.
- **Vs [`mask`](../mask/):** unrelated — `mask` is a Makefile-
  style task runner whose name collides with the data-masking
  domain. The two solve completely different problems and the
  name overlap is an accident.
- **Vs `pg_dump` + a hand-written `sed` / `awk` pipeline:** the
  hand-rolled approach falls apart on JSON / ARRAY / `bytea` /
  `tsvector` columns where naive text substitution corrupts the
  encoding; on multi-column dependencies (a fake email must
  match the fake first/last name); and on referential integrity
  (fake the `users.id` and every FK becomes orphaned).
  greenmask understands the pg_dump TOC + COPY stream and
  routes transformations through type-aware functions.

## Caveats

- **Postgres only.** greenmask is built on `pg_dump` /
  `pg_restore`, so the source and destination are both
  Postgres. MySQL / MariaDB / SQL Server / Oracle anonymization
  needs a different tool (e.g. `myanon` for MySQL).
- **Logical dump, not physical.** greenmask uses `pg_dump`
  semantics (logical, row-by-row), not `pg_basebackup` (block-
  level, much faster on large DBs but not transformable).
  Expect dump times comparable to a normal `pg_dump -j N`,
  which on a 500 GB prod DB is hours, not minutes. The trade is
  intentional — you cannot anonymize block-level dumps.
- **Transformer choice is your responsibility.** A `Hash`
  transformer preserves uniqueness but not distribution; a
  `ReplaceWithFaker` transformer preserves distribution shape
  but not uniqueness; a `NoiseFloat` transformer preserves
  aggregates but lets the reader narrow the range. Pick per
  column based on what the dev / staging workflow needs to
  remain *true* and what the privacy posture needs to remain
  *false*. Greenmask gives you the primitives — the policy is
  yours.
- **Foreign-key integrity needs deterministic transformers.**
  If a `users.id` is hashed, every `orders.user_id` that
  references it must be hashed with the same seeded function
  or the FKs become invalid on restore. Use the same `Hash`
  transformer + the same seed for matched columns; greenmask
  supports this but it is configuration discipline, not
  automatic.
- **Validation is opt-in.** `greenmask validate` is a separate
  step that diffs pre- and post-transformation samples to
  catch "you wrote `column: emial` and it silently did
  nothing" typos. Wire it into CI; do not trust a green dump
  without a validate step.
- **Active development.** v0.x version line — expect breaking
  config changes between minor versions. Pin the binary
  version in CI and re-validate the config after upgrades.
