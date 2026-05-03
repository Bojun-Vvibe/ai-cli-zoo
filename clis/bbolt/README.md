# bbolt

> **The maintained successor to BoltDB — an embedded
> pure-Go key/value store, plus a CLI for
> inspecting, repairing, and benchmarking the
> on-disk file**. The library powers etcd's storage
> engine; the CLI (`bbolt`) lets you crack open any
> `*.db` file produced by it: list buckets, dump
> keys, walk the B+tree, check page integrity,
> compact, and benchmark write throughput. If you
> have a Go service writing to a Bolt file and
> something looks wrong, `bbolt` is the answer to
> "what is actually in there". Pinned to **v1.4.3**
> ([LICENSE](https://github.com/etcd-io/bbolt/blob/main/LICENSE),
> MIT).

Source: <https://github.com/etcd-io/bbolt>

## TL;DR

BoltDB (`boltdb/bolt`) was the canonical Go embedded
KV store — a single mmap'd file, ACID transactions,
B+tree, no server. Upstream went unmaintained in
2018; etcd forked it as `etcd-io/bbolt` and has
been the de-facto maintainer ever since (etcd
itself stores all its raft / mvcc state in a Bolt
file). Most Go agents you'll touch — etcd, k3s
embedded etcd, Consul snapshots, Trivy / Grype
caches, Caddy's storage, BadgerDB-alternative
projects — write a Bolt file under the hood. When
"why is this 12 GB" or "why does startup hang on
this file" comes up in production, you reach for
the `bbolt` CLI.

## Install

```bash
# Go install (any platform with Go ≥ 1.22)
go install go.etcd.io/bbolt/cmd/bbolt@v1.4.3

# Homebrew (community formula)
brew install bbolt   # if available in your tap; otherwise use go install

# Build from source
git clone https://github.com/etcd-io/bbolt
cd bbolt && git checkout v1.4.3
make build && cp bin/bbolt /usr/local/bin/

# verify
bbolt version    # bbolt Version: 1.4.3
```

`bbolt` is a single static Go binary; the same
binary embeds both the `bbolt` library and the CLI
subcommands.

## Use it for

```bash
# What buckets exist in this file?
bbolt buckets path/to/my.db

# What keys are in a bucket? (root-level)
bbolt keys path/to/my.db my-bucket

# Dump a value for inspection
bbolt get path/to/my.db my-bucket my-key

# Walk every page in the file (B+tree structure, freelist, meta)
bbolt page path/to/my.db 0      # page 0 = meta
bbolt pages path/to/my.db       # summary of all pages

# Integrity check — scans every page, validates checksums
bbolt check path/to/my.db

# Statistics — page counts, allocation, leaf vs branch
bbolt stats path/to/my.db

# Compact (reclaim freelist pages, shrink file)
bbolt compact -o compacted.db source.db

# Benchmark write throughput against a fresh DB
bbolt bench --count 1000000 --batch-size 1000

# Surgery: list every bucket recursively, including nested
bbolt buckets path/to/my.db --depth all

# Dump with pretty hex (binary keys / values)
bbolt get -format hex path/to/my.db raft state
```

The CLI is read-mostly by design — `compact` is the
one mutation, and it always writes to a *new* file
so the source is preserved. Repairing a corrupted
Bolt file means: `compact` to a fresh path, verify
with `check`, then atomically swap.

## Why include it in a CLI catalog

1. **It is the universal forensic tool for any Go
   service that stores state on disk.** etcd, k3s'
   embedded etcd, Consul (raft snapshots), Caddy
   (cert / lock storage), Trivy (vuln cache),
   Grype, Bleve indexes, NATS JetStream KV
   backends, dozens of small daemons — all write
   Bolt files. Without `bbolt` the file is opaque;
   with it you can confirm what the service
   actually persisted, separate "data missing"
   from "code path didn't run", and recover keys
   from a half-corrupted file by `compact`-ing
   into a clean copy.
2. **It is the canonical reference for "embedded
   KV store" in Go.** When evaluating
   alternatives ([`badger`](https://github.com/dgraph-io/badger),
   `pebble`, BoltDB-API-compatible forks like
   `bbolt-rust`), you want to know what the
   incumbent does — `bbolt page` and `bbolt stats`
   show you the B+tree layout, freelist behavior,
   and write-amplification characteristics
   directly. Reading is faster than reading a
   paper.
3. **`bbolt bench` gives you a fair micro-benchmark
   for the same hardware your service will run on.**
   Synthetic write throughput against a fresh DB,
   tunable batch size, sequential vs random keys —
   exactly the right number to plug into "is the
   bottleneck Bolt or my code". Beats guessing
   from upstream marketing benchmarks.

For an LLM-CLI workflow, `bbolt` is the answer when
your local agent's persistent cache (rate limits,
embeddings index, conversation history) lives in a
Bolt file and you need to inspect or surgically
edit it without booting the agent process.

## Vs Already Cataloged

- **Vs [`sqlite`](../sqlite/) / [`litecli`](../litecli/):**
  different storage model — SQLite is relational
  with a SQL surface; Bolt is a B+tree of
  bucket→key→value byte blobs. SQLite wins for
  ad-hoc queries and joins; Bolt wins for "I just
  want a typed map persisted, transactionally,
  with no server". `bbolt` CLI is the equivalent
  of `sqlite3` for Bolt files: open, peek, dump,
  benchmark.
- **Vs [`etcdctl`](../etcdctl/):** different
  abstraction layer — `etcdctl` talks to a running
  etcd cluster via gRPC; `bbolt` opens the
  underlying *file* directly. When etcd won't
  start ("snapshot mismatch", "db not found"), you
  cannot use `etcdctl` — you need `bbolt page` /
  `bbolt check` on the on-disk `member/snap/db`.
- **Vs [`badger`](https://github.com/dgraph-io/badger)
  / [`pebble`](https://github.com/cockroachdb/pebble):**
  competing libraries — Badger and Pebble are
  LSM-tree-based, optimized for write-heavy / SSD
  workloads. Bolt is B+tree-based, optimized for
  read-heavy / mostly-static workloads with
  occasional small writes. Pick by access pattern;
  use the corresponding library's own inspector
  CLI.
- **Vs [`redis-cli`](../redis-cli/):** different
  deployment model — Redis is a server, Bolt is
  embedded. The right comparison is "how do I
  inspect persisted state for a tool that didn't
  ship a CLI of its own": Redis ships `redis-cli`,
  Bolt-using tools rely on the generic `bbolt`
  CLI.

## Caveats

- **Single-writer.** Bolt allows many readers and
  exactly one writer at a time. `bbolt` CLI takes
  a *read* lock by default — safe to run against a
  live DB. `compact` and `bench` need exclusive
  access; do them against a *copy* of the file
  while the service runs against the original.
- **mmap'd files inherit the OS's mmap quirks.** On
  32-bit systems the file size is capped near 2 GB.
  On all systems, `du -h` is misleading because
  Bolt over-allocates pages to the freelist; use
  `bbolt stats` for the truth.
- **No SQL.** No joins, no secondary indexes, no
  range queries beyond a single bucket's key
  ordering. If your inspection needs `WHERE
  value LIKE '%foo%'`, dump to JSON and pipe
  through [`jq`](../jq/) / [`jp`](../jp/).
- **Format compatibility:** `bbolt` v1.4 reads files
  written by older `boltdb/bolt` and earlier
  `bbolt` versions. Going *forward* (writing with
  v1.4, reading with an old `boltdb/bolt`) is
  also safe because the on-disk format is frozen.
- **MIT license** — permissive; safe to embed both
  the library and the CLI in proprietary
  distributions with attribution.
