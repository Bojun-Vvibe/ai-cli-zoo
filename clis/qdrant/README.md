# qdrant

> Snapshot date: 2026-04. Upstream: <https://github.com/qdrant/qdrant>
> License file: <https://github.com/qdrant/qdrant/blob/master/LICENSE>
> Pinned: `v1.17.1` (2026-03-27, server). The server is the source of
> truth; the official clients (`pip install qdrant-client`, `npm
> install @qdrant/js-client-rest`, `cargo add qdrant-client`,
> `go get github.com/qdrant/go-client`) track minor versions in step.
> Default branch is `master`, not `main` — minor papercut for
> drive-by contributors.

A **production-shaped vector search engine written in Rust**. Server-
hosted gRPC + REST API, pluggable storage (in-memory / mmap / on-disk
RocksDB), distributed mode with sharding + replication, payload
filtering as a first-class query primitive (not a post-filter), and
the `Q-PQ` quantisation family (scalar, product, binary) for shrinking
billion-vector tables to fit in RAM. The contrast point against
[`lancedb`](../lancedb/) is "library vs. engine": you `docker run -p
6333:6333 qdrant/qdrant` instead of importing a Python package, get a
process with its own RBAC + snapshots + cluster membership, and the
clients are thin RPC stubs in every language.

## 1. Install footprint

- **Server (everyday shape)**: `docker run -p 6333:6333 -p 6334:6334
  -v $(pwd)/qdrant_storage:/qdrant/storage qdrant/qdrant:v1.17.1`
  — image is ~80 MB, RSS at idle ~60 MB, REST on 6333, gRPC on 6334,
  web UI at `http://localhost:6333/dashboard`.
- **Clients**:
  - Python: `pip install qdrant-client` (~15 MB, pulls `grpcio` +
    `httpx` + `pydantic`).
  - Node: `npm install @qdrant/js-client-rest` (REST-only, ~200 KB)
    or `@qdrant/js-client-grpc` for the streaming surface.
  - Rust: `cargo add qdrant-client` — the upstream crate, gRPC-native.
  - Go: `go get github.com/qdrant/go-client`.
  - Java / Kotlin / .NET: official Maven / NuGet packages.
- **Local dev without Docker**: `pip install qdrant-client` and use
  `QdrantClient(":memory:")` for an in-process embedded mode (good
  for tests + CI fixtures, not for production — single-process, no
  persistence guarantees).
- **Cluster mode**: same Docker image, set `QDRANT__CLUSTER__ENABLED=true`
  and seed peer URLs via env; Raft for membership, configurable shard
  count + replication factor per collection.
- Workspace footprint on disk: `qdrant_storage/` holds one
  subdirectory per collection (segments + WAL + snapshots). Add to
  `.gitignore` unless you intend to commit a small reference corpus.

## 2. License

Apache-2.0 (file is `LICENSE` at the repo root,
<https://github.com/qdrant/qdrant/blob/master/LICENSE>). Whole server
+ all official clients are Apache-2.0 — no ELv2 / SSPL / BSL games.
Qdrant Cloud is the commercial managed offering; the OSS server has
no feature gating.

## 3. Models supported

Qdrant is a **pure vector index + payload store** — model axis is
"which embedder produces the vectors you POST". No bundled embedder;
the engine accepts any dimensionality (typical 384 / 768 / 1024 /
1536 / 3072 / 4096) and any distance (`Cosine` / `Dot` / `Euclid` /
`Manhattan`). Sparse vectors (BM25-shaped, `name: "text", indices: [...],
values: [...]`) are first-class alongside dense, and the `query` API
fuses both with RRF or DBSF in one round-trip.

For "embedder lives in the same process" workloads, the `fastembed`
Python package (same upstream) ships ONNX-quantised sentence-
transformers (`BAAI/bge-small-en-v1.5`, `intfloat/multilingual-e5-
small`, `nomic-embed-text-v1.5`) plus image embedders (`Qdrant/clip-
ViT-B-32`); `client.add(collection, documents=["..."])` then embeds
client-side and uploads in one call. This is the closest the OSS
shape gets to [`marqo`](../marqo/)'s server-side embed contract —
embedding still happens in the client process, not the engine, but
you avoid wiring a separate embedder service.

## 4. MCP support

**Official MCP server** (`mcp-server-qdrant`,
<https://github.com/qdrant/mcp-server-qdrant>) — exposes `qdrant-
store` and `qdrant-find` tools so MCP-aware agents
([`opencode`](../opencode/), [`claude-code`](../claude-code/),
[`crush`](../crush/), [`fast-agent`](../fast-agent/)) can write +
recall semantic memories without touching the REST API. Run via
`uvx mcp-server-qdrant` or as a Docker sidecar; configure with
`QDRANT_URL` + `COLLECTION_NAME` + `EMBEDDING_MODEL` (FastEmbed
identifier).

The base server also exposes a clean OpenAPI spec at `/openapi.json`,
so any "MCP-from-OpenAPI" bridge (e.g. `openapi-mcp-generator`)
gives you a tool surface over the full collections / points / search
API in one command.

## 5. Sub-agent model

None — Qdrant is a substrate. Multi-agent / multi-process access is
the everyday case: writers and readers coexist via the WAL +
segment design, snapshots give you point-in-time backups, and the
distributed mode handles N writer + N reader topologies via Raft +
sharding. Multitenancy is first-class via the `group_id` payload
field + tenant-key index — cheaper than collection-per-tenant and
the recommended pattern past ~100 tenants.

For composite retrieval pipelines, the new `query` API (`POST
/collections/{c}/points/query`) chains "prefetch (ANN over a
quantised index) → rescore (full-precision) → group_by (one result
per `doc_id`)" inside one round-trip — the engine-side counterpart
to a small reranking sub-agent, no second network hop.

## 6. Telemetry stance

**Opt-in anonymous usage telemetry** is on by default in the OSS
server (process count, version, OS, collection count — no payload
data, no query content). Disable by setting
`QDRANT__TELEMETRY_DISABLED=true` (env) or `telemetry_disabled: true`
in `config.yaml`. The `/telemetry` REST endpoint shows you exactly
what would be sent if enabled — useful for security review.

Data plane has no egress: all collection data lives on the disk
volume you mount, all queries terminate in the engine, and the
optional Qdrant Cloud is a separate hosted product reached via a
different URL + API key.

For sensitive corpuses, set `service.api_key` in `config.yaml` to
require a bearer token on every request, run behind your own TLS-
terminating reverse proxy, and enable JWT-based RBAC (1.9+) for
per-collection / read-only / payload-restricted tokens.

## 7. Prompt-cache strategy

**Not a prompt cache** — different layer. But Qdrant's segment +
mmap design gives you a "warm working set" effect for free: hot
segments (recently-queried + frequently-queried) stay in the OS
page cache, so a workload of "10 K queries with overlapping ANN
neighbourhoods" warms the relevant HNSW graph nodes and steady-
state p99 lands in single-digit milliseconds on commodity NVMe.
Quantisation (binary or scalar) shrinks the on-RAM index 4–32×,
so a billion-vector table can stay resident on a 64 GB box where
the unquantised version would page constantly.

For "cache the LLM call that *uses* the retrieval result", layer
[`gptcache`](../gptcache/) on top of the LLM call site — orthogonal,
both cache hits compose multiplicatively to cut total latency.

## 8. Hot keybinds

No TUI; the everyday surfaces are the Python client and the web
dashboard at `http://localhost:6333/dashboard` (collection browser,
point inspector, query playground, cluster view).

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct, Filter, FieldCondition,
    MatchValue, SparseVectorParams, Prefetch, Fusion, FusionQuery,
)

# 1. Connect
client = QdrantClient(url="http://localhost:6333", api_key=None)

# 2. Create a collection — dense + sparse + payload index
client.create_collection(
    collection_name="docs",
    vectors_config={"dense": VectorParams(size=384, distance=Distance.COSINE)},
    sparse_vectors_config={"bm25": SparseVectorParams()},
)
client.create_payload_index(
    collection_name="docs", field_name="tenant_id",
    field_schema="keyword", tenant=True,
)

# 3. Upsert
client.upsert(
    collection_name="docs",
    points=[
        PointStruct(
            id=1,
            vector={"dense": [0.1] * 384, "bm25": {"indices": [42, 99], "values": [1.2, 0.8]}},
            payload={"text": "...", "tenant_id": "alice", "source": "wiki"},
        ),
    ],
)

# 4. Hybrid query (dense prefetch + sparse prefetch → RRF fusion → top-10)
results = client.query_points(
    collection_name="docs",
    prefetch=[
        Prefetch(query=[0.1] * 384, using="dense", limit=50),
        Prefetch(query={"indices": [42, 99], "values": [1.2, 0.8]}, using="bm25", limit=50),
    ],
    query=FusionQuery(fusion=Fusion.RRF),
    query_filter=Filter(must=[FieldCondition(key="tenant_id", match=MatchValue(value="alice"))]),
    limit=10,
)
```

Index types worth knowing:
- HNSW (default for dense) — `m=16, ef_construct=128` is the
  out-of-box; raise `ef` at query time for higher recall.
- Quantisation — `quantization_config=ScalarQuantization(scalar=
  ScalarQuantizationConfig(type="int8", always_ram=True))` for the
  4× memory cut with negligible recall loss; `BinaryQuantization`
  for the 32× cut on high-dim cosine vectors (works best ≥1024-d).
- Sparse — Qdrant indexes sparse vectors via inverted index, so
  BM25-shaped retrieval scales the same way Lucene does.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A Rust-native vector engine that treats payload
filtering, sparse vectors, multitenancy, and quantisation as
first-class query primitives, not bolt-ons**. The new `query` API
chains prefetch → rescore → fusion → group_by in one round-trip,
so a hybrid (dense + sparse) retrieval with tenant filter, binary-
quantisation prefetch, full-precision rescore, and "one result per
doc_id" grouping is one HTTP call — not a client-side pipeline of
three calls + dedupe code. Cluster mode (Raft + sharding +
replication) is in the OSS image with no feature gate, and the
official clients exist for Python / Node / Rust / Go / Java /
Kotlin / .NET.

**Weakness.** **Server to operate** — you own the Docker image, the
disk volume, the snapshot rotation, the TLS, the API-key
distribution, and the cluster Raft quorum. The "no DB, just a
directory" simplicity of [`lancedb`](../lancedb/) is gone; in
exchange you get a process designed for the multi-tenant + multi-
writer + billion-vector shape that an embedded DB cannot reach.
The default `master` branch (not `main`) trips up automation that
assumes `main`. Distributed-mode rebalancing during shard moves is
manual (you trigger `replicate_shard` / `move_shard` by hand) —
fine at the scales most people hit, surprising if you came from
Elasticsearch's auto-rebalance.

**When to choose.**

- You need a **server-shaped vector store** with its own RBAC,
  snapshots, TLS, and cluster membership — i.e. the operations
  contract is "we run a database", not "we ship a directory".
- You need **first-class hybrid retrieval** (dense + sparse + payload
  filter + grouping + reranking) in one round-trip, with the engine
  doing the fusion server-side.
- You're at **multitenant scale** (hundreds to millions of tenants)
  and want one collection + tenant-keyed payload index, not
  collection-per-tenant.
- You need **quantisation that actually works** — binary + scalar +
  product, with `always_ram=true` to pin the quantised index in
  RAM and `rescore=true` to rescue recall on the top-K.
- You want **language-neutral clients** (Python / Node / Rust / Go
  / Java / .NET / Kotlin all official) so a polyglot stack hits the
  same engine without per-language ports.

**When not to choose.** You want **no server to run** —
[`lancedb`](../lancedb/) is the embedded library shape and removes
the Docker image entirely. You want **server-side embedding** (POST
text or image URL, get retrieval results back without ever
generating a vector client-side) — [`marqo`](../marqo/) bakes the
embedder into the engine, Qdrant does not. You only need a **side-
table inside an existing Postgres** — `pgvector` is simpler and
lives in the DB you already operate. You want **fully managed
serverless with zero infra to think about** — Pinecone / Turbopuffer
are closer to that shape, though Qdrant Cloud is the official
managed alternative when you want the same engine + zero ops.
