# kine

> **kine** — k3s-io/kine's etcd-shim that lets a Kubernetes API
> server run against a SQL backend (SQLite, PostgreSQL, MySQL/MariaDB,
> NATS JetStream) instead of an etcd cluster. Pinned to **v0.14.16**,
> Apache-2.0 — license file:
> [LICENSE](https://github.com/k3s-io/kine/blob/master/LICENSE).

Source: <https://github.com/k3s-io/kine>

## TL;DR

`kine` (Kine Is Not Etcd) speaks the etcd v3 gRPC API on the front
and a SQL `INSERT INTO kine ...` revision log on the back. The
Kubernetes apiserver thinks it is talking to etcd; under the hood
every key/value/lease/watch maps to rows in a `kine` table that
SQLite, Postgres, MySQL, or NATS JetStream owns. It is the storage
backend k3s ships with by default — `k3s server` is `kube-apiserver
+ kine + sqlite` in one binary — and can be lifted out and run as a
standalone daemon in front of any control plane.

The reason to care: **operating etcd correctly is a job**. Quorum
sizing, periodic defragmentation, snapshot restore, certificate
rotation, "the cluster is wedged because one of three members
crashed and the other two have clock drift" — all of that
disappears when the backing store is `Postgres-on-RDS` (HA + PITR
already solved by the DBA team) or `sqlite-on-disk` (tiny single-
node cluster). The trade-off is etcd's first-class lease + watch
semantics get re-implemented over SQL polling, which costs CPU at
high churn rates but is invisible at single-cluster scale.

## Install

```bash
# Single static Go binary — the canonical path is the GitHub release
# https://github.com/k3s-io/kine/releases/tag/v0.14.16

# Build from source
go install github.com/k3s-io/kine@v0.14.16

# Bundled inside k3s — no separate install needed there
curl -sfL https://get.k3s.io | sh -
```

## Example commands

```bash
# Run kine against a local SQLite file (the k3s default)
kine --endpoint "sqlite:///var/lib/kine/state.db"

# Run against PostgreSQL — the apiserver now stores everything in PG
kine --endpoint "postgres://kine:secret@db.local:5432/kine?sslmode=disable"

# Run against MySQL / MariaDB
kine --endpoint "mysql://kine:secret@tcp(db.local:3306)/kine"

# Run against NATS JetStream (event-sourced backend, k3s 1.28+)
kine --endpoint "jetstream://nats.local:4222"

# Point a stock kube-apiserver at it
kube-apiserver --etcd-servers=http://127.0.0.1:2379 ...
```

## Niche it occupies

**SQL-backed etcd shim for Kubernetes control planes** — different
niche from the rest of the catalog. Closest neighbours:

- [`bbolt`](../bbolt/) — embedded key-value store (etcd's own
  underlying engine). Pick `kine` when the goal is to avoid running
  etcd-the-cluster; pick bbolt when you want the same KV shape
  embedded in your own Go process.
- [`dolt`](../dolt/) / [`rqlite`](../rqlite/) /
  [`litefs`](../litefs/) / [`litestream`](../litestream/) —
  distributed / replicated SQLite shapes. Orthogonal: those make
  SQLite itself HA, kine sits *above* SQLite (or PG) and exposes the
  etcd surface a Kubernetes apiserver expects.
- [`k3s`](https://k3s.io) / [`k0sctl`](../k0sctl/) /
  [`kind`](../kind/) / [`minikube`](../minikube/) — bundled
  Kubernetes distros. Pick kine when you want to swap *only* the
  storage backend of an existing distro (or your own); the distros
  already use it where appropriate (k3s embeds it, k0s can be
  configured to).
- [`vcluster`](../vcluster/) — virtual Kubernetes clusters running
  inside a parent cluster. vcluster uses kine internally (SQLite by
  default, optional Postgres/MySQL) for exactly this reason — every
  virtual cluster gets its own SQL-backed control plane without
  spinning up an etcd cluster per tenant.

Pairs cleanly with [`pgcat`](../pgcat/) / [`pgbackrest`](../pgbackrest/)
(Postgres pooling + PITR for the kine database when you choose the PG
backend) and with [`velero`](../velero/) (cluster-state backup that
becomes "snapshot the SQL database" on a kine-backed cluster — much
simpler than etcd snapshot/restore choreography).

Caveats: not a drop-in for *every* etcd consumer (kine implements the
v3 KV/lease/watch surface kube-apiserver needs, not the full etcd API
— skuba/Vault/Patroni/Calico-typha that depend on etcd-specific verbs
will not work against it); watch-heavy workloads (operators that
LIST+WATCH on every CRD churn) put more load on the SQL polling loop
than equivalent etcd traffic; PostgreSQL backend wants a recent (≥13)
server because kine relies on `LISTEN/NOTIFY` for change propagation.

## Citation

- Repo: <https://github.com/k3s-io/kine>
- Latest release: **v0.14.16**
- License: **Apache-2.0**
- License file: [LICENSE](https://github.com/k3s-io/kine/blob/master/LICENSE)
