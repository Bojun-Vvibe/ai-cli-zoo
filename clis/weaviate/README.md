# weaviate

> Snapshot date: 2026-04. Upstream: <https://github.com/weaviate/weaviate>
> License file: <https://github.com/weaviate/weaviate/blob/main/LICENSE>
> Pinned: `v1.37.2` (2026-04-23, server). The Weaviate server is
> the source of truth; the official clients (`pip install
> weaviate-client`, `npm install weaviate-client`,
> `go get github.com/weaviate/weaviate-go-client/v5`,
> `cargo add weaviate-community` for community Rust, plus official
> Java) track minor versions in step.

A **vector + keyword + hybrid search engine written in Go** with
a **modular embedder + reranker + generator pipeline baked into the
server** — the `text2vec-*`, `multi2vec-*`, `reranker-*`, and
`generative-*` module families let you POST raw text or image URLs
to Weaviate and have it embed, search, rerank, and (optionally)
synthesise an answer in one round-trip, server-side. The contrast
point against [`qdrant`](../qdrant/) is "modules vs. raw engine":
Qdrant is deliberately payload-store + vector-index only and
expects you to embed client-side; Weaviate ships the embedder +
reranker + RAG-synthesis layer as pluggable modules so the client
can stay thin.

## 1. Install footprint

- **Server (everyday shape)**: `docker run -p 8080:8080 -p 50051:50051
  -e AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED=true
  -e PERSISTENCE_DATA_PATH=/var/lib/weaviate
  -v $(pwd)/weaviate_data:/var/lib/weaviate
  cr.weaviate.io/semitechnologies/weaviate:1.37.2`
  — image is ~120 MB, RSS at idle ~150 MB, REST on 8080, gRPC on
  50051. Adding modules (e.g. `text2vec-transformers`,
  `reranker-cohere`, `generative-openai`) is via the
  `ENABLE_MODULES` env var + `DEFAULT_VECTORIZER_MODULE`.
- **Embedded mode**: the Python client supports `EmbeddedOptions()`
  which downloads the binary to `~/.cache/weaviate-embedded` and
  spawns it as a child process — useful for tests + notebooks, not
  for production.
- **Clients**:
  - Python: `pip install weaviate-client` (v4.x, gRPC-native, ~30 MB).
  - Node / TS: `npm install weaviate-client` (v3.x, gRPC-native).
  - Go: `go get github.com/weaviate/weaviate-go-client/v5`.
  - Java: `io.weaviate:client` on Maven Central.
  - Community: Rust (`weaviate-community`), Ruby, .NET.
- **Cluster mode**: same Docker image, set `CLUSTER_HOSTNAME` +
  `CLUSTER_GOSSIP_BIND_PORT` + `CLUSTER_DATA_BIND_PORT` and seed
  via `CLUSTER_JOIN`; Raft for schema metadata (added in 1.25),
  per-class sharding + replication factor, async replication with
  read-repair.
- **Workspace footprint on disk**: `weaviate_data/` holds one
  subdirectory per shard with HNSW + inverted index segments +
  WAL. Add to `.gitignore` unless committing a small reference
  corpus.

## 2. License

BSD-3-Clause (file is `LICENSE` at the repo root,
<https://github.com/weaviate/weaviate/blob/main/LICENSE>). The
core server, all official client libraries, and all bundled
modules in the main repo are BSD-3-Clause — no ELv2 / SSPL / BSL
games. Weaviate Cloud (WCD) is the commercial managed offering;
the OSS server has no feature gating, including cluster mode +
multi-tenancy + replication.

## 3. Models supported

Weaviate is a **vector store + module host**. The vector axis is
provider-agnostic (any dimensionality, distances `cosine` / `dot` /
`l2-squared` / `manhattan` / `hamming`), and the module families
plug in actual model providers:

- **Vectorizer modules** (server-side embedding at index + query
  time): `text2vec-openai`, `text2vec-cohere`, `text2vec-huggingface`,
  `text2vec-aws` (Bedrock), `text2vec-google` (Vertex / AI Studio),
  `text2vec-jinaai`, `text2vec-voyageai`, `text2vec-mistral`,
  `text2vec-databricks`, `text2vec-ollama` (point at a local
  [`ollama`](../ollama/) instance), `text2vec-transformers`
  (server-side container running a HuggingFace model), `text2vec-
  contextionary` (the original bundled embedder), and `multi2vec-*`
  variants for image / audio / video.
- **Reranker modules**: `reranker-cohere`, `reranker-voyageai`,
  `reranker-jinaai`, `reranker-transformers`, `reranker-nvidia`.
- **Generator modules** (RAG synthesis on top of search results):
  `generative-openai`, `generative-anthropic`, `generative-cohere`,
  `generative-google`, `generative-aws`, `generative-mistral`,
  `generative-ollama`, `generative-databricks`, `generative-
  friendliai`, `generative-nvidia`, `generative-xai`.
- **BYO vectors** is also fully supported — declare the class with
  `vectorizer: "none"` and POST the float arrays you produced
  client-side, exactly the [`qdrant`](../qdrant/) shape.

## 4. MCP support

**Official MCP server** (`mcp-server-weaviate`,
<https://github.com/weaviate/mcp-server-weaviate>) exposes
`weaviate-collection-create`, `weaviate-objects-insert`,
`weaviate-objects-query`, and `weaviate-objects-hybrid-search`
tools so MCP-aware agents ([`opencode`](../opencode/),
[`claude-code`](../claude-code/), [`crush`](../crush/),
[`fast-agent`](../fast-agent/)) can read + write semantic memories
without touching the gRPC API. Run via `uvx mcp-server-weaviate`
or as a Docker sidecar; configure with `WEAVIATE_URL` +
`WEAVIATE_API_KEY` + collection-name patterns. The base server
also exposes a clean OpenAPI + GraphQL spec, so any
"MCP-from-OpenAPI" bridge gives you a tool surface over the full
schema / objects / search API.

## 5. Sub-agent model

None — Weaviate is a substrate. Multi-tenant access is
first-class: the **multi-tenancy** mode (`multiTenancyConfig:
{ enabled: true }` on a class) creates per-tenant shards
on-demand, with `autoTenantCreation` for write-time provisioning
and `autoTenantActivation` for cold-tenant warmup; a single
collection can host hundreds of thousands of tenants without the
`one collection per tenant` overhead, with per-tenant snapshot +
deactivate + delete operations as first-class API calls. For
composite retrieval pipelines, the GraphQL `Hybrid` operator
fuses BM25 + vector with `alpha` (RRF or relative-score fusion)
in one query, optionally followed by `rerank` (one of the reranker
modules) and `generate` (one of the generator modules) — the
"prefetch → rescore → fuse → rerank → synthesise" pipeline runs
inside the engine, not the client.

## 6. Telemetry stance

Weaviate's OSS server sends **anonymous usage telemetry**
(version, OS, module list, object count buckets — no payload
data, no query content) on by default. Disable by setting
`DISABLE_TELEMETRY=true` in the server env. Module endpoints (for
non-local modules like `text2vec-openai`) egress to the configured
provider — that's the obvious one to remember when scoping data
boundaries; `text2vec-transformers` + `reranker-transformers` +
`generative-ollama` give you a fully local pipeline if egress is
a hard constraint.

For sensitive corpuses, set `AUTHENTICATION_APIKEY_ENABLED=true`
+ `AUTHENTICATION_APIKEY_ALLOWED_KEYS=...` to require API-key
auth, enable RBAC (1.29+) for per-collection / per-tenant
read/write/admin roles, run behind your own TLS reverse proxy, and
use OIDC (`AUTHENTICATION_OIDC_ENABLED=true`) for human + service
identity federation.

## 7. Prompt-cache strategy

**Not a prompt cache** — but the **generative module pipeline**
caches embeddings server-side for a query batch (the embedder
module is called once per unique input within a request, not per
result), and the HNSW + inverted index segments stay warm in OS
page cache the same way [`qdrant`](../qdrant/) does. PQ
(`vectorIndexConfig: { pq: { enabled: true, segments: 96 } }`)
shrinks the on-RAM index ~8× with negligible recall loss; BQ
(binary quantisation) shrinks it 32× for high-dim cosine vectors.

For "cache the LLM call that *uses* the retrieval result", the
`generative-*` modules don't cache responses themselves — layer
[`gptcache`](../gptcache/) or [`helicone`](../helicone/) at the
LLM call site, orthogonal to Weaviate.

## 8. Hot keybinds

No TUI; the everyday surfaces are the Python client and the
Weaviate Cloud / self-hosted Console at `http://localhost:8080`
(GraphQL playground, schema browser, query builder).

```python
import weaviate
from weaviate.classes.config import Configure, Property, DataType
from weaviate.classes.query import Filter, MetadataQuery, HybridFusion

# 1. Connect
client = weaviate.connect_to_local(
    host="localhost", port=8080, grpc_port=50051,
)

# 2. Create a collection — server-side OpenAI embedder + Cohere reranker
client.collections.create(
    name="Docs",
    vectorizer_config=Configure.Vectorizer.text2vec_openai(
        model="text-embedding-3-small",
    ),
    reranker_config=Configure.Reranker.cohere(model="rerank-v3.5"),
    generative_config=Configure.Generative.openai(model="gpt-4o-mini"),
    properties=[
        Property(name="text", data_type=DataType.TEXT),
        Property(name="tenant_id", data_type=DataType.TEXT),
        Property(name="source", data_type=DataType.TEXT),
    ],
    multi_tenancy_config=Configure.multi_tenancy(
        enabled=True, auto_tenant_creation=True,
    ),
)

# 3. Write — server embeds at insert time
docs = client.collections.get("Docs").with_tenant("alice")
docs.data.insert({"text": "Refunds are 30 days...", "source": "wiki"})

# 4. Hybrid query → rerank → generate, in one round-trip
results = docs.query.hybrid(
    query="how long is the refund window",
    alpha=0.5,                     # 0=BM25 only, 1=vector only
    fusion_type=HybridFusion.RELATIVE_SCORE,
    limit=10,
    rerank=weaviate.classes.query.Rerank(
        prop="text", query="refund window",
    ),
    return_metadata=MetadataQuery(score=True, distance=True),
)

# 5. Or do RAG synthesis server-side
answer = docs.generate.hybrid(
    query="how long is the refund window",
    grouped_task="Answer the user's question using only the docs above.",
    limit=5,
)
print(answer.generated)
```

Index types worth knowing:
- HNSW (default for dense) — `efConstruction=128, ef=-1` is the
  out-of-box; raise `ef` at query time for higher recall.
- Flat — exact (no approximation), good for ≤10K-vector collections.
- Dynamic — starts flat, transitions to HNSW after a threshold.
- Quantisation — PQ (`pq.enabled=true`), BQ (`bq.enabled=true`,
  `rescore=true`), SQ (`sq.enabled=true`).
- Inverted index for BM25 + payload filtering — first-class,
  composes with vector in `hybrid()`.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **The vectorizer + reranker + generator
modules let the engine handle the full retrieval-augmented
pipeline server-side**, so a Python client can do
`docs.generate.hybrid(query=..., grouped_task=...)` and get an
LLM-synthesised answer over BM25-fused + vector-search +
reranked context in one network round-trip — no separate
embedder service, no separate reranker call, no separate LLM
call from the client. Multi-tenancy as a first-class shard model
(per-tenant HNSW + inverted indexes, auto-create + auto-activate
+ deactivate + snapshot per tenant) scales to hundreds of
thousands of tenants under one collection without the
"collection-per-tenant" antipattern. BSD-3-Clause across the
whole engine + every official module + every official client.

**Weakness.** **The module abstraction is leaky** when you want
fine-grained control. Switching from `text2vec-openai` to
`text2vec-cohere` is a class-config change *and* a re-embed of
all existing data (the vectors live in the index by the
old model's geometry); cross-encoder reranker latency adds 50–
200 ms per query depending on `top_k` + provider; the GraphQL
surface (`hybrid` / `nearVector` / `rerank` / `generate`) is more
to learn than Qdrant's REST. Cluster mode pre-1.25 had famous
"split-brain on schema change" footguns; 1.25+'s Raft schema
metadata fixed the worst of it but you still need to read the
upgrade notes carefully across major versions. Server-side
embedding means provider keys + per-class quotas + provider
outages all live inside Weaviate's blast radius — fine when the
trade is "thin client", less fine when you want strict client-
side control.

**When to choose.**

- You want a **server-shaped vector engine that also handles
  embedding + reranking + RAG synthesis** in the engine, so the
  client SDK stays thin and polyglot consumers all hit the same
  pipeline.
- You're at **multi-tenant scale** (hundreds of thousands of
  tenants) and want per-tenant shards + auto-provisioning +
  per-tenant snapshots as first-class API calls, not a
  "collection-per-tenant" antipattern.
- You want **hybrid (BM25 + vector + rerank + generate)** as one
  GraphQL call, with provider choice as a class-config knob.
- You want **straight BSD-3-Clause across the whole stack**,
  including cluster mode + multi-tenancy, with no enterprise
  feature gating in the OSS image.

**When not to choose.** You want **a payload-store + vector-index
without the embedder + reranker + generator modules** —
[`qdrant`](../qdrant/) is the leaner shape, you embed client-
side. You want **no server to run** — [`lancedb`](../lancedb/) is
embedded library, no Docker. You want **the bundled embedder *and*
a Docker image with everything in 2 GB** — [`marqo`](../marqo/)'s
"inference + vector store + REST in one container" is the closer
match. You only need a **side-table inside an existing Postgres**
— `pgvector` is simpler and lives in the DB you already operate.
You want **fully managed serverless with zero infra to think
about** — Weaviate Cloud (WCD) is the official managed shape, or
look at Pinecone / Turbopuffer for serverless-first vendors.
