# milvus

> Snapshot date: 2026-04. Upstream: <https://github.com/milvus-io/milvus>
> Pinned release: `v2.6.15` (HEAD `7fc234d1fe8200a0d48e2f905c67647fc084711c`).
> License file: [`LICENSE`](https://github.com/milvus-io/milvus/blob/master/LICENSE)
> (sha `261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64`, Apache-2.0).

`milvus` is a high-performance, cloud-native open-source vector
database written in Go (with C++ kernels for the index code
paths) — a sibling to [`qdrant`](../qdrant/),
[`weaviate`](../weaviate/), [`chroma`](../chroma/), and
[`lancedb`](../lancedb/) in the catalog's vector-store row,
biased toward **multi-tenant, partitioned, billions-of-vectors
production** workloads where a single collection has hundreds
of millions of rows and the deploy shape is Kubernetes, not a
laptop. The CLI surface in scope here is the operator side:
the `milvus` server binary, the `milvus_cli` Python TUI, the
`birdwatcher` debugging shell, plus `bulk-writer` /
`bulk-import` for offline ingestion. From the agent / RAG
catalog's perspective, Milvus is a backing store the rest of
the tools talk to over an OpenAI-style retrieval API or
through the language SDKs (`pymilvus`, `milvus-node`,
`milvus-rust`).

## 1. Install footprint

- Standalone (single-node, embedded etcd + MinIO):
  `bash -c "$(curl -sfL
  https://raw.githubusercontent.com/milvus-io/milvus/master/scripts/standalone_embed.sh)"
  start` brings the server up on `:19530` (gRPC) / `:9091`
  (HTTP) in a single container.
- Distributed (production): Helm chart
  `milvus/milvus-helm` deploys the proxy / coord / data-node /
  query-node / index-node tiers with pluggable etcd, Kafka /
  Pulsar (WAL), and S3 / MinIO / Azure Blob (object store).
- Embedded: **Milvus Lite** — `pip install pymilvus[lite]` —
  runs the same query engine in-process against a local SQLite
  file, API-compatible with the server, useful for unit tests
  and local development.
- Operator CLI: `pip install milvus-cli` installs an
  interactive shell (`milvus_cli`) with TAB-completion against
  a running cluster.

## 2. Repo, version, license

- Repo: <https://github.com/milvus-io/milvus>
- Latest release: `v2.6.15` (2026-04-17).
- HEAD pinned at this snapshot:
  `7fc234d1fe8200a0d48e2f905c67647fc084711c`.
- License: Apache-2.0. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/milvus-io/milvus/blob/master/LICENSE)
  (sha `261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64`).

## 3. What it actually does

The data model is **collection → partition → shard → segment**,
each segment a sealed columnar file (vectors + scalar fields)
with one or more index types attached. Index families
supported in 2.6.x: `FLAT`, `IVF_FLAT`, `IVF_SQ8`, `IVF_PQ`,
`HNSW`, `DISKANN` (on-disk for >RAM corpora), `SCANN`,
`GPU_IVF_FLAT` / `GPU_IVF_PQ` / `GPU_BRUTE_FORCE` /
`GPU_CAGRA` (NVIDIA RAFT-backed), plus sparse-vector indexes
(`SPARSE_INVERTED_INDEX`, `SPARSE_WAND`) for BM25 / SPLADE
hybrid retrieval. Query primitives:

- **Vector search** — `search(...)` ANN over one anns-field,
  with metric (`L2`, `IP`, `COSINE`, `JACCARD`, `HAMMING`).
- **Hybrid search** — `hybrid_search(...)` fuses N anns
  searches with a `WeightedRanker` or `RRFRanker` reranker;
  one query can mix dense + sparse + scalar filters.
- **Scalar filtering** — boolean expressions over JSON /
  varchar / int / float fields, applied as a pre-filter or
  post-filter to the ANN scan.
- **Range search** — return all vectors within a similarity
  threshold rather than a top-k.
- **Iterator API** — paginated stable scans over very large
  result sets without recomputing the full top-k.
- **Bulk import** — Parquet / JSON / numpy file → segment
  files written directly to object store, skipping the
  insert-path proxy bottleneck for one-shot loads of billions
  of rows.

## 4. MCP support

None first-party. Milvus is a database; an MCP-aware host
agent reaches it through the language SDKs or through a
RAG-shaped wrapper ([`r2r`](../r2r/), [`ragflow`](../ragflow/),
[`docsgpt`](../docsgpt/), [`khoj`](../khoj/),
[`local-deep-research`](../local-deep-research/),
[`paper-qa`](../paper-qa/)) which itself may publish an MCP
server. Community MCP servers exist as small `pymilvus`
adapters but are not part of the upstream.

## 5. Sub-agent model

N/A — vector database, not an agent. The relevant primitive is
**resource groups + query nodes**: a single cluster can
isolate "high-QPS production retrieval" from "batch reindex"
into separate node pools, with rate limits and cache tiers per
group, which is the multi-tenant story most catalog
laptop-grade vector stores skip.

## 6. Telemetry stance

Off in OSS. The server emits Prometheus `/metrics` for opt-in
scraping; nothing phones home to milvus.io. The hosted Zilliz
Cloud is opt-in via separate signup. Embedded Milvus Lite is
fully local — the SQLite file is the only artifact.

## 7. Token / context strategy

N/A. The context-window analogue here is the **resource model
behind index types**: HNSW and IVF families fit in RAM and
saturate latency; DiskANN trades cold latency for serving
billions of vectors per node from NVMe; GPU indexes
(`GPU_CAGRA`, `GPU_IVF_PQ`) saturate throughput at the cost of
GPU residency. Picking the index is the cost / latency / RAM
trilemma every Milvus deploy resolves once.

## 8. Hot keybinds

`milvus_cli` is a `prompt-toolkit`-shaped REPL — `↑` / `↓`
history, `TAB` completion against schema and collection names,
`Ctrl-C` cancel, `Ctrl-D` exit. The server itself is a
non-interactive long-running daemon.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **DiskANN + GPU index families behind one
storage-compute-disaggregated cluster, with hybrid dense +
sparse + scalar retrieval as a first-class query** — that
combination is rare in OSS. DiskANN lets a single query node
serve a billion 768-dim vectors from NVMe with millisecond
latency without paying for a billion-vector RAM box; GPU
indexes hit hundreds of thousands of QPS on an H100; sparse
inverted indexes mean BM25 / SPLADE join the same query
engine instead of living in a separate Elastic; resource groups
isolate workloads. For a RAG corpus that has graduated past
"5M vectors fits on my laptop", Milvus's deploy curve is
flatter than the alternatives because every dimension
(throughput, RAM, on-disk, GPU, multi-tenant) has an answer
inside the same cluster.

**Weakness.** **Operational surface is large.** The
distributed deploy involves a coord tier, a query / data /
index node tier, etcd for metadata, a Kafka / Pulsar WAL, and
S3 / MinIO for object storage — the Helm chart is a real
opinion you have to engage with, and version upgrades touch
several components in a coordinated dance. A team that just
wants "a vector store that works" is better served by a
single-binary peer ([`qdrant`](../qdrant/),
[`weaviate`](../weaviate/)) until scale forces the move.

**Choose `milvus` when** the corpus is past 50M vectors, the
deploy target is Kubernetes, the workload mixes high-QPS
retrieval with periodic batch reindex, hybrid (dense + sparse +
scalar) is a hard requirement, or DiskANN / GPU indexes are
load-bearing. **Choose something else when** the corpus fits
in one box and the priority is "minutes to first query" — pick
[`qdrant`](../qdrant/) or [`weaviate`](../weaviate/) for a
single-binary server, [`chroma`](../chroma/) or
[`lancedb`](../lancedb/) for embedded / file-shaped storage,
or [`sqlite-vec`](../sqlite-vec/) when "a SQLite extension"
is the entire integration budget.
