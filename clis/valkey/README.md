# valkey

- **Repo:** https://github.com/valkey-io/valkey
- **Version:** 9.0.3 (tagged 2026-02-24; latest stable on the 9.0 line, includes CVE-2025-67733 / CVE-2026-21863 / CVE-2026-27623 fixes)
- **License:** BSD-3-Clause — see [`COPYING`](https://github.com/valkey-io/valkey/blob/unstable/COPYING). The project remains fully BSD-licensed and is the Linux Foundation–stewarded open-source continuation of the Redis 7.2 codebase, forked in March 2024 after Redis Inc. moved upstream Redis to the dual RSALv2 / SSPLv1 source-available license.
- **Language:** C
- **Install:** `brew install valkey` · `apt install valkey-server` (Debian trixie / Ubuntu 24.10+) · `dnf install valkey` · official Docker images at `valkey/valkey` · binaries are `valkey-server`, `valkey-cli`, `valkey-benchmark`, `valkey-check-aof`, `valkey-check-rdb`, `valkey-sentinel`

## Overview

Valkey is a single-binary, in-memory data-structure server
that speaks the **RESP3** wire protocol and is drop-in
compatible with existing Redis 7.2 clients, Sentinel
deployments, and Cluster topologies. It keeps the original
Redis data model (strings, lists, hashes, sets, sorted sets,
streams, hyperloglog, bitmap, geo, JSON via the `valkey-json`
module, vector search via `valkey-search`), the same RDB +
AOF persistence story, the same Lua + Functions scripting
surface, and the same Pub/Sub and keyspace-notification
mechanics. What it adds since the fork: multi-threaded
asynchronous I/O on by default, RDMA support, dual-channel
replication that decouples the snapshot stream from the
command stream (so a full sync no longer back-pressures a
busy primary), Hash Field TTL, and an active community of
contributors from AWS, Google, Oracle, Ericsson, and others
governed under the Linux Foundation rather than a single
vendor.

## Niche

**Vendor-neutral, fully-open-source in-memory KV / data-
structure server with a Redis-compatible wire protocol**.
The role is the same one Redis OSS used to occupy: a
microsecond-latency cache, session store, rate-limiter,
queue, leaderboard, pub/sub bus, or tiny-document store —
just under a license that distros, cloud providers, and
internal platforms can ship without RSALv2 / SSPLv1 friction.

## When to use

- You need a Redis-protocol-compatible server but want a
  permissively licensed (BSD-3) build that Debian, Fedora,
  Alpine, and the major clouds all package.
- Existing Redis client libraries / Sentinel / Cluster
  tooling must keep working unchanged — Valkey is a drop-in
  on the wire and on disk (RDB and AOF formats are
  compatible).
- You want the multi-threaded I/O and dual-channel
  replication wins from the post-fork roadmap without
  changing application code.
- You need a process the rest of your local dev stack can
  point at: `valkey-server --save '' --appendonly no
  --port 6379` boots a stateless cache in one command, and
  every Redis SDK in every language already speaks to it.
- You want a project where the governance is a foundation,
  not a single vendor — relevant for procurement and for
  long-term roadmap risk.

## When NOT to use

- You actively want Redis Inc.'s newer commercial features
  (Redis Stack proprietary modules tied to recent versions,
  Redis Cloud control plane, Redis Enterprise active-active
  CRDTs) — those live with upstream Redis under RSALv2 /
  SSPLv1 and are not in Valkey.
- You need a disk-resident KV with multi-TB datasets per
  node — Valkey is in-memory first, with persistence as
  durability, not as primary storage. Reach for RocksDB,
  FoundationDB, or ScyllaDB for that shape.
- You want a SQL surface — use Postgres / SQLite / DuckDB.
- You need a strongly-consistent KV with linearizable reads
  out of the box — Valkey Cluster gives partitioned
  availability with eventual consistency on failover; use
  etcd / Consul / FoundationDB if linearizability is
  load-bearing.
- A simpler embedded KV would do (single process, no
  network) — use SQLite / BoltDB / sled / RocksDB embedded.

## Comparison vs alternatives in zoo

- [`sqlite-utils`](../sqlite-utils/) — embedded SQL store;
  use SQLite when the data is durable-by-default and on
  disk, Valkey when it lives in RAM and is accessed by many
  processes over the network.
- [`duckdb`](../duckdb/) — embedded analytic engine over
  Parquet / CSV / Arrow; orthogonal — Valkey is OLTP-shaped
  state, DuckDB is OLAP-shaped scans.
- [`litestream`](../litestream/) (if present) — durability
  layer for SQLite over object storage; different problem
  shape.
- [`nats`] / [`rabbitmq`] (not in the zoo as of this entry) —
  message brokers; Valkey can do Pub/Sub and Streams but is
  not a guaranteed-delivery broker. Pick a real broker if
  durable queuing semantics are load-bearing.

## Why it earns a slot in an AI-native workflow

Agent loops accumulate state quickly: tool-call traces,
embedding caches, scratchpad memory, rate-limit counters,
LLM-response caches keyed on prompt hash, vector search
indexes, per-session conversation history. Valkey covers
all of those with one process: `SET prompt:<hash>
<response> EX 86400` for a 24-hour LLM cache; `XADD trace
* tool=<name> args=<json>` to append tool-call events to a
stream and consume them with consumer groups; `HSET
session:<id> ...` for per-session memory; `valkey-search`
for HNSW vector indexes when you need approximate
nearest-neighbor lookup but don't yet need a dedicated
vector DB. Because the wire protocol is RESP, every existing
agent framework's Redis adapter works unchanged — you swap
the `redis://` URL for the Valkey endpoint and ship.

## Example invocations

```bash
# Boot a local cache: no persistence, port 6379, single thread
valkey-server --save '' --appendonly no --port 6379

# Boot a persistent instance with AOF, on a non-default port
valkey-server --port 6380 --appendonly yes --appendfsync everysec \
              --dir ./valkey-data

# Talk to it
valkey-cli -p 6380 SET hello world EX 60
valkey-cli -p 6380 GET hello
valkey-cli -p 6380 INFO replication

# Pub/Sub
valkey-cli SUBSCRIBE chan &
valkey-cli PUBLISH chan 'hello world'

# Streams (append-only log with consumer groups)
valkey-cli XADD events '*' kind page url /home
valkey-cli XGROUP CREATE events workers '$' MKSTREAM
valkey-cli XREADGROUP GROUP workers w1 COUNT 10 BLOCK 0 STREAMS events '>'

# Built-in load test
valkey-benchmark -t set,get -n 100000 -q -p 6380

# Sentinel (HA failover) on a separate config
valkey-sentinel /etc/valkey/sentinel.conf
```
