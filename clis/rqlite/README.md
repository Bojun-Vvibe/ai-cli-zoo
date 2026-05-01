# rqlite

> **Distributed, fault-tolerant SQLite** — a single Go binary that
> wraps SQLite in a Raft consensus layer and an HTTP/SQL API, giving
> you a relational database with strong consistency, leader
> election, and snapshotting on top of the same embedded SQLite
> engine you already trust. Bring 3 (or 5) nodes up, point your app
> at any of them, and survive a node loss without losing a write.
> Pinned to **v10.0.1** (commit
> `5607879b88fb9ab9ed3b667b71c34c0172cac164`,
> [LICENSE](https://github.com/rqlite/rqlite/blob/v10.0.1/LICENSE),
> MIT).

Source: <https://github.com/rqlite/rqlite>

## TL;DR

`rqlite` answers the awkward question "I love SQLite but I need
**more than one machine** and I can't lose writes when one dies."
Each node runs the `rqlited` daemon, which embeds SQLite plus a
HashiCorp Raft log; a cluster (typically 3 or 5 nodes) elects a
leader, replicates every write through the Raft log, and applies
committed entries to its local SQLite file. Reads can be served
at `none` (any node, fastest, possibly stale), `weak` (leader,
no quorum check), `linearizable` (leader confirms it is still
leader via heartbeat), or `strong` (read goes through the Raft
log alongside writes — full linearizable consistency at the cost
of a round trip). Clients talk plain HTTP/JSON: `POST /db/execute`
for writes, `POST /db/query` for reads, `POST /db/request` for
mixed batches and parameterised statements; an interactive
`rqlite` shell ships in the same binary for `psql`-style ad-hoc
sessions. Operationally it is a single static Go binary — no JVM,
no external coordinator, no separate WAL service — with on-disk
snapshots, automatic compaction, TLS for both the HTTP API and
inter-node Raft traffic, optional basic-auth + permission file,
backup/restore via `GET /db/backup` (binary SQLite file) and
`POST /db/load`, and a read-only "non-voter" node mode for
geo-distributed read replicas. It is **not** a sharded
multi-writer system: total write throughput is capped by the
leader's local SQLite, so think "small-to-medium relational data
that must not lose writes during a node failure", not "horizontally
scalable OLTP".

## Install

```bash
# pre-built binary (Linux / macOS / Windows, amd64 + arm64)
curl -L https://github.com/rqlite/rqlite/releases/download/v10.0.1/rqlite-v10.0.1-darwin-arm64.tar.gz \
  | tar xz -C /tmp
sudo mv /tmp/rqlite-v10.0.1-darwin-arm64/{rqlited,rqlite,rqbench} /usr/local/bin/

# Homebrew
brew install rqlite

# Docker (official image)
docker run -p 4001:4001 rqlite/rqlite:8

# verify
rqlited -version          # v10.0.1
```

## Examples

```bash
# 1. Single-node sandbox: bring up rqlited, create a table, insert + query
rqlited -node-id 1 ~/rqlite-data &
curl -XPOST 'http://127.0.0.1:4001/db/execute?pretty&timings' -H 'Content-Type: application/json' -d '[
  "CREATE TABLE foo (id INTEGER NOT NULL PRIMARY KEY, name TEXT)",
  "INSERT INTO foo(name) VALUES (\"fiona\")"
]'
curl -G 'http://127.0.0.1:4001/db/query?pretty&level=strong' \
  --data-urlencode 'q=SELECT * FROM foo'

# 2. Three-node cluster on one host (ports 4001/4003/4005), with the second
#    and third nodes joining the first as the bootstrap leader
rqlited -node-id 1 -http-addr localhost:4001 -raft-addr localhost:4002 ~/data1 &
rqlited -node-id 2 -http-addr localhost:4003 -raft-addr localhost:4004 \
  -join http://localhost:4001 ~/data2 &
rqlited -node-id 3 -http-addr localhost:4005 -raft-addr localhost:4006 \
  -join http://localhost:4001 ~/data3 &
rqlite -H localhost -p 4001               # interactive psql-style shell
```
