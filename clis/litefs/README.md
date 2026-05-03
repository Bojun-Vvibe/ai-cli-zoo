# litefs

> **Distributed SQLite via FUSE-level page replication** — a single
> Go binary that mounts a FUSE file system in front of a SQLite
> database, captures every page write at the VFS layer (no
> per-row triggers, no logical decoding), and ships the resulting
> *LTX* (Lite Transaction) files to followers over HTTP. One
> writer node + N read replicas, with primary election via
> Consul / static config / lease, and a transparent proxy that
> forwards writes from a follower to the current primary so the
> application can pretend it has a single local SQLite. Pinned to
> **v0.5.14** (release published 2025-04-22,
> [LICENSE](https://github.com/superfly/litefs/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/superfly/litefs>

## TL;DR

`litefs` is what you reach for when you love SQLite's "the
database is a file in your process" ergonomics but you have
finally outgrown one machine — you need a hot read-replica in
another region, you need crash-recovery beyond "restore
yesterday's `.db` from S3," or you want zero-downtime restarts.
It is a *page-level* replication layer, not a logical one: the
follower file is a byte-identical copy of the primary's pages, so
there is no "schema must match" gotcha and no per-statement
overhead. The trade is that all writes funnel through one
primary at a time (it is a single-writer system, not a Raft
multi-master) and replication lag is measured in tens of
milliseconds, not zero. For read-heavy SQLite workloads that
need HA + horizontal read scaling, this is the smallest hop from
"single-file SQLite" to "actually distributed."

## Install

```bash
# Homebrew (macOS / Linux)
brew install litefs

# from a release tarball (Linux only — FUSE)
curl -L https://github.com/superfly/litefs/releases/download/v0.5.14/litefs-v0.5.14-linux-amd64.tar.gz | tar xz
sudo install litefs /usr/local/bin/

# Docker (the supported deployment shape)
docker pull flyio/litefs:0.5.14

# from source
git clone https://github.com/superfly/litefs && cd litefs
go build ./cmd/litefs
sudo install litefs /usr/local/bin/

# verify
litefs version    # litefs v0.5.14
```

`litefs` is Linux-only at runtime (it depends on FUSE); on macOS
you can build the binary but you cannot mount, which is why most
deployments are containerised.

## License

Apache-2.0 — see [LICENSE](https://github.com/superfly/litefs/blob/main/LICENSE).
Permissive with patent grant; redistribute and embed freely with
attribution.

## One Concrete Example

```bash
# 1. minimal litefs.yml: mount /var/lib/litefs over /litefs, leader via static config
cat > /etc/litefs.yml <<'YAML'
fuse:
  dir: "/litefs"
data:
  dir: "/var/lib/litefs"
proxy:
  addr: ":20202"
  target: "localhost:8080"      # the app process
  db: "app.db"
lease:
  type: "static"
  advertise-url: "http://node1:20202"
  candidate: true               # this node can be primary
  promote: true
YAML

# 2. start litefs as PID 1 of the container; it execs the app once mounted
litefs mount -- /usr/local/bin/myapp --db /litefs/app.db

# 3. application opens SQLite as if it were local — writes go through the FUSE layer,
#    are captured as LTX, and shipped to followers
sqlite3 /litefs/app.db "INSERT INTO events VALUES (1, 'hello');"

# 4. on a follower, the same path mirrors the primary's bytes within ~10-50 ms
sqlite3 /litefs/app.db "SELECT * FROM events;"   # reads served locally

# 5. a follower can still accept writes — the litefs proxy forwards them to the
#    current primary transparently (the app sees a normal POST at :20202 -> :8080)
curl -X POST http://follower:20202/events -d '{"id":2,"msg":"world"}'

# 6. observe replication lag and primary identity
curl http://localhost:20202/db/app.db/info | jq
# {
#   "name": "app.db",
#   "primary": "node1",
#   "tx_id": 4128,
#   "lag_ms": 14,
#   ...
# }
```

## Niche It Fills

**SQLite that survives a node loss without giving up SQLite.**
Postgres / MySQL / CockroachDB / [`rqlite`](../rqlite/) /
[`dolt`](../dolt/) / [`turso-cli`](../turso-cli/) all distribute
relational data, but each makes you adopt a new server, a new
wire protocol, or a new operational story. SQLite-as-a-file is
unbeatable for embedded use, single-binary deployment, and
"the database is a backup-able artefact" workflows; `litefs`
keeps that exact shape and adds replication + failover at the
page layer. Same `sqlite3` CLI, same `WAL` mode, same `.dump`
backup story — plus a hot read replica two regions away.

## Why use it

Three things `litefs` does that the obvious "just use Postgres"
answer does not:

1. **No application change.** The app opens
   `/litefs/app.db` exactly like it would open
   `/var/data/app.db`; SQL semantics are identical because the
   underlying engine *is* SQLite. No new driver, no connection
   string, no Postgres-flavoured SQL adjustments. A Rails / Django
   / Flask / Phoenix / Go app developed against local SQLite
   moves to multi-region read-replica HA by changing the mount
   path.
2. **Page-level replication, not row-level.** Capturing writes at
   the VFS layer means LTX files are byte-identical page diffs;
   replicas apply them atomically per transaction. No "schema
   drift" class of bugs, no per-row trigger overhead, no logical
   decoding pipeline to babysit. A 10 GB SQLite file replicates
   as ~the same volume of bytes you wrote, full stop.
3. **Built-in proxy hides the single-writer constraint.** Real
   applications have writes coming from any pod, not just the
   one currently elected primary. The litefs HTTP proxy on each
   follower forwards write-shaped requests to the current primary
   transparently, so app code does not need to know who the
   leader is. Combined with `lease.type: "consul"` for automatic
   failover, this gives you a "single SQLite that happens to be
   highly available" abstraction.

For an LLM agent stack that uses SQLite as the per-tenant
state store ([`marimo`](../marimo/) / [`langfuse`](../langfuse/)
/ [`pocketbase`](../pocketbase/) etc. all default to SQLite),
`litefs` is the smallest path from "lives on one box" to "survives
a redeploy and serves reads from a closer region."

## Vs Already Cataloged

- **Vs [`litestream`](../litestream/):** sister project, different
  goal — `litestream` is a *streaming backup* tool by the same
  author (continuous WAL shipping to S3 / GCS / SFTP / SSH); it
  recovers a single primary from object storage but does not
  serve live reads. `litefs` is replication + read-scale +
  failover; `litestream` is point-in-time-restore. Many setups
  run both: `litefs` for HA + read replicas, `litestream` for
  the off-site backup tape.
- **Vs [`rqlite`](../rqlite/):** orthogonal tradeoff — `rqlite`
  is Raft-replicated SQLite where every node is a Raft member
  and writes go through consensus (slower writes, stronger
  consistency, no FUSE). `litefs` is async page replication
  (faster writes, eventual consistency on followers, requires
  FUSE). Pick `rqlite` when you need linearizable writes across
  nodes; pick `litefs` when you need the fastest possible local
  writes plus async read scale.
- **Vs [`dolt`](../dolt/):** very different shape — `dolt` is
  Git-for-data with branching, diff, merge, and a MySQL wire
  protocol; `litefs` is plain SQLite with replication. Choose
  `dolt` for collaborative data workflows; choose `litefs` for
  operational HA on a normal app DB.
- **Vs [`turso-cli`](../turso-cli/):** related lineage — Turso
  is a hosted multi-tenant fork of SQLite (libSQL) with global
  edge replicas as a managed service; `litefs` is the
  self-hosted equivalent you operate on your own boxes. Pick
  Turso when you want managed; pick `litefs` when you need to
  run on-prem or want zero vendor lock.
- **Vs `pocketbase` / `marimo` / `langfuse` (all cataloged):**
  these are *consumers* of SQLite; mount their data dir on
  `litefs` and they gain HA without code changes.

## Caveats

- **Linux + FUSE only.** macOS hosts cannot mount; develop locally
  on plain SQLite, deploy under `litefs` on Linux containers.
  Windows is not supported.
- **Single-writer at a time.** `litefs` is *not* multi-master.
  All writes funnel through the current primary; followers proxy
  writes back to it. If the primary is unreachable, the system
  cannot accept writes until a new primary is elected (lease
  TTL, typically 10–30 s with Consul). Read availability is
  unaffected.
- **Replication is asynchronous.** Followers lag the primary by
  tens of milliseconds in the same DC, and by tens to hundreds
  of milliseconds across regions. Reads after a write may not
  see the write yet; if you need read-your-writes consistency,
  read from the primary or pin the session.
- **No transactions across databases.** `litefs` replicates one
  SQLite file at a time; ATTACH DATABASE across two `litefs`
  files does not get a single distributed transaction.
- **FUSE has overhead vs raw filesystem.** Synthetic
  micro-benchmarks show ~5–15 % write overhead vs SQLite on
  ext4. For most application workloads (mixed read-heavy) this
  is invisible; for write-saturated benchmarks it is measurable.
- **Operational complexity grows with HA.** Lease backend
  (Consul / etcd / static), primary election, network reachability
  between nodes, FUSE mount lifecycle, container PID 1 ordering
  (`litefs mount -- app`) all become things you have to know.
  Single-box `litestream` is operationally cheaper if you only
  need backup, not HA.
