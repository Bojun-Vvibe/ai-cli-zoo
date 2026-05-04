# tigerbeetle

- **Upstream:** https://github.com/tigerbeetle/tigerbeetle
- **Version:** v0.17.3 (latest stable as of 2026-05-05)
- **License:** Apache-2.0 — [LICENSE](https://github.com/tigerbeetle/tigerbeetle/blob/main/LICENSE)

## What it does

`tigerbeetle` is a single statically-linked Zig binary that bundles a
purpose-built distributed **financial accounting database** plus its
operator CLI. The binary covers every lifecycle verb — `tigerbeetle format`
provisions a fixed-size data file, `tigerbeetle start --addresses=...`
runs a replica that participates in a deterministic Viewstamped-Replication
cluster (typically 3 or 6 replicas for strict serializability under
fail-stop and network-partition faults), `tigerbeetle benchmark` drives
the cluster with a workload generator, and `tigerbeetle repl` opens an
interactive shell that speaks the binary protocol over a Unix socket and
accepts the two domain verbs the database exposes — `create_accounts` and
`create_transfers` (debit-credit double-entry, with linked-transfer chains
for atomic multi-leg postings, pending/post/void two-phase transfers, and
balance invariants enforced at insert time, not by a trigger). The
on-disk format is one append-only file per replica, every write is
checksummed and replicated before acknowledgement, and recovery is bit-for-bit
deterministic against a captured input log — the same property that makes
the test harness reproduce production bug reports from a recorded seed.

## Why it's interesting / niche

The "OLTP" databases in the existing zoo (`duckdb`, `litecli`, `pgcli`,
`harlequin`, `usql`, `datasette`, `dolt`, `rqlite`, `turso-cli`, `valkey`)
are general-purpose SQL or KV stores; *none* of them are domain-specific
double-entry ledgers, and none expose a deterministic-replication CLI as a
first-class verb. TigerBeetle is the niche of "the database is the
business logic" — for payments, exchange ledgers, billing, in-game
economies, ad-spend accounting, anywhere the schema is "accounts and
transfers between them with strict invariants" — and the CLI is shaped
around that single domain (no `CREATE TABLE`, no SQL, just `format` /
`start` / `repl` with two verbs). It pairs naturally with the rest of
the zoo: feed event traces to `duckdb` for analytics, expose balance
queries through `datasette`, run schema-shaped reconciliation in
`harlequin` against a Postgres mirror, while TigerBeetle owns the
authoritative ledger that must never lose a cent.

The deterministic-replay design has an AI-adjacent angle too — an agent
that proposes a refactor of accounting code can replay a recorded
production trace through the new code path locally and verify zero
divergence before the change is shipped, the same property that
`temporal workflow replay` provides for workflows.
