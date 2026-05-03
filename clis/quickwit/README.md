# quickwit

> **Cloud-native distributed search engine purpose-built for
> append-only log, trace, and event data on object storage.**
> Decoupled compute / storage architecture: indexes are tantivy
> segments written directly to S3 / GCS / Azure Blob / MinIO, and
> stateless searcher pods fan out range-scoped reads across them
> — no hot replicas, no JVM, no per-node disk planning. Pinned
> to **v0.8.2**
> ([LICENSE](https://github.com/quickwit-oss/quickwit/blob/main/LICENSE),
> Apache-2.0; version checked via `gh api repos/quickwit-oss/quickwit/releases/latest`).

Source: <https://github.com/quickwit-oss/quickwit>

## TL;DR

`quickwit` is a Rust search engine that treats the object store
as the source of truth. An indexer writes immutable
`tantivy`-format split files (the same engine that powers
`meilisearch`'s lower layers) directly to S3-compatible
storage; a metastore (PostgreSQL or a file in the same bucket)
tracks split metadata; stateless searcher nodes load only the
byte ranges they need to answer a query. The result is a system
that scales storage and query independently, costs roughly the
same as keeping the raw logs in S3 (because that's basically
what it does), and serves both an Elasticsearch-compatible
`_search` API and OTLP/Jaeger trace ingestion out of the same
binary.

## Install

```bash
# Single static binary (Linux / macOS / aarch64)
curl -L https://install.quickwit.io | sh
./quickwit/quickwit run

# Container
docker run --rm -p 7280:7280 quickwit/quickwit run

# Kubernetes (Helm)
helm repo add quickwit https://helm.quickwit.io
helm install quickwit quickwit/quickwit \
  --set config.storage.s3.bucket=my-logs
```

## Example

```bash
# Create a log index from a YAML schema
quickwit index create --index-config ./logs.yaml

# Ingest newline-delimited JSON
cat events.ndjson | quickwit index ingest --index logs --input-path -

# Query via the ES-compatible search API
curl 'http://localhost:7280/api/v1/logs/search?query=severity:ERROR+AND+service:checkout'

# Or via SQL-ish quickwit query language
quickwit index search --index logs \
  --query 'severity:ERROR' --max-hits 50 --start-timestamp 1714000000
```

## When to use

- You have terabytes-to-petabytes of append-only log / trace
  data and want full-text + structured search without paying
  for hot-replica storage.
- You already have an OpenTelemetry pipeline and want a
  Jaeger-compatible trace backend that doesn't require
  Cassandra or Elasticsearch operations.
- You want an Elasticsearch DSL drop-in for log search but
  refuse to run a stateful Elastic / OpenSearch cluster.

## When NOT to use

- You need general-purpose document search with frequent
  updates / deletes (use `meilisearch` / Elasticsearch /
  OpenSearch — quickwit is append-mostly).
- Your data fits comfortably on one box and `ripgrep` over a
  rotated log directory already answers every question you
  have.
- You need true real-time (sub-second from ingest to
  searchable) — quickwit's commit cadence is on the order of
  tens of seconds, by design.

## Orthogonality vs existing zoo entries

- **vs [`loki`-style stack / `vector` / `fluent-bit`]** —
  those are log *shippers* and (for Loki) a label-indexed log
  store. Quickwit is the *search engine* on the cold tier:
  point a `vector` sink at it for full-text indexing on top of
  the same S3 bucket Loki / Mimir / Tempo already use.
- **vs [`opensearch` / Elasticsearch]** — same ES `_search`
  surface, completely different cost model: stateless
  searchers + S3 vs hot-replica JVM cluster. Quickwit is what
  you reach for when the ES bill outweighs the freshness
  advantage.
- **vs [`duckdb`](../duckdb/) / [`datasette`](../datasette/)
  / [`q-text-as-data`](../q-text-as-data/)** — those are
  ad-hoc analytical SQL over files on disk. Quickwit is a
  long-running indexed search service over a continuously
  growing object-store corpus, with sub-second p99 on
  needle-in-haystack queries that a full table scan cannot
  match.
- **vs [`jaeger`](../jaeger/) backend choices** — quickwit
  *is* a supported Jaeger storage backend (since 0.7) and can
  consolidate logs + traces in one engine, removing the
  separate Cassandra / ES dependency Jaeger normally needs.

## Niche / tags

`search` · `logs` · `traces` · `observability` · `s3-native` ·
`rust` · `tantivy`
