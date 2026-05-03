# pgcat

> **PostgreSQL pooler + proxy with sharding, load balancing,
> failover and read/write split — written in Rust.** Speaks the
> Postgres wire protocol (drop-in for `pgbouncer`-shaped client
> configs), but adds primary/replica routing, query parsing to
> auto-route reads vs. writes, mirroring (shadow traffic to a
> second cluster), per-shard pools, and SCRAM auth — all from a
> single static binary with a hot-reloadable TOML config.
> Pinned to **v1.2.0**
> ([LICENSE](https://github.com/postgresml/pgcat/blob/main/LICENSE),
> MIT).

Source: <https://github.com/postgresml/pgcat>

## TL;DR

Point your application at `pgcat` on `:6432` instead of Postgres
on `:5432`. `pgcat` opens a fixed pool to each backend
(`primary`, `replicas[*]`), parses every incoming statement to
classify it as read or write, sends writes to `primary` and load-
balances reads across `replicas` (round-robin / random /
least-outstanding), supports session and transaction pooling
modes per pool, and reloads `pgcat.toml` on `SIGHUP` without
dropping in-flight clients. For sharded setups it hashes a
configured key (e.g. `tenant_id`) and routes each connection to
the matching shard pool. Mirroring duplicates writes to a
secondary cluster for zero-downtime migrations.

## Install

```bash
# Container (the upstream-recommended path)
docker run --rm -p 6432:6432 \
  -v "$PWD/pgcat.toml:/etc/pgcat/pgcat.toml:ro" \
  ghcr.io/postgresml/pgcat:latest

# From source (Rust 1.70+)
git clone https://github.com/postgresml/pgcat && cd pgcat
cargo build --release
./target/release/pgcat pgcat.toml

# Helm chart (Kubernetes)
helm repo add postgresml https://postgresml.github.io/pgcat
helm install pgcat postgresml/pgcat -f values.yaml
```

## Example

```bash
# Start with a config file (reads pools, shards, auth from TOML)
pgcat /etc/pgcat/pgcat.toml

# Hot-reload after editing pgcat.toml — no client disconnects
kill -HUP $(pgrep pgcat)

# Connect via psql like it's plain Postgres
PGPASSWORD=app psql -h 127.0.0.1 -p 6432 -U app -d shard_0
```

## When to use

- You outgrew `pgbouncer` and need read/write split, sharding,
  or replica failover handled at the pool layer instead of in
  application code.
- You are migrating Postgres clusters and want mirrored writes
  to a new primary while the old one still serves reads.
- You want a single Rust binary with structured logs and
  Prometheus metrics on `:9930` instead of a C daemon plus
  sidecars.

## When NOT to use

- You only need transaction pooling for one Postgres instance —
  `pgbouncer` is smaller and battle-tested for that single
  niche.
- Your driver already does client-side read/write split (e.g.
  Rails `replica:` config) and you do not want a second hop.
- You need MySQL or another engine — pgcat is Postgres-only.

## Niche / tags

`db-tool` · `postgres` · `connection-pooler` · `sharding` ·
`rust`
