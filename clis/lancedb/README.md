# lancedb

> Snapshot date: 2026-04. Upstream: <https://github.com/lancedb/lancedb>
> License file: <https://github.com/lancedb/lancedb/blob/main/LICENSE>
> Pinned: `v0.28.0-beta.9` (2026-04-19, Node/Rust line). Python tracks
> a parallel cadence (`pip install lancedb` resolves to a 2026-04
> release on PyPI). The Rust core + Python / Node / Java / Go
> bindings move together; pin the exact PyPI / npm version since
> the on-disk Lance v2 format is still tightening minor details
> across betas.

A **developer-friendly embedded retrieval library for multimodal
AI**. Vector + full-text + hybrid search over the Lance columnar
on-disk format (Apache-Arrow-shaped, mmap-friendly, versioned).
"Embedded" means there is **no server to run** — `lancedb.connect(
"./data")` opens a directory of Lance datasets like SQLite opens
a `.db` file, query latency is sub-millisecond on million-row tables
because the index is mmap'd and the storage is columnar Arrow.

The everyday surface is the Python SDK (`import lancedb`); the same
on-disk format is queryable from Rust, Node, Java, Go bindings, and
the OSS DuckDB / Polars connectors.

## 1. Install footprint

- **Python**: `pip install lancedb` (or `uv pip install lancedb`).
  Pulls `pyarrow`, `pylance`, the Rust core via `maturin`-built
  wheels for Linux/macOS/Windows on x86_64 + arm64. ~80 MB
  installed.
- **Node**: `npm install @lancedb/lancedb` — ships prebuilt
  `napi-rs` binaries for the same matrix.
- **Rust**: `cargo add lancedb` — the upstream crate.
- **Java**: Maven coordinates `com.lancedb:lancedb`.
- **Go**: `go get github.com/lancedb/lancedb-go`.
- Python 3.9+ / Node 18+. No daemon, no separate process; the
  database **is** the directory you point it at.
- Workspace footprint: a `lancedb/` directory holding one
  subdirectory per table, each a Lance v2 dataset (data files +
  manifest + indices subfolder). Add to `.gitignore` unless you
  intend to commit a small reference index.
- Optional embeddings extras:
  `pip install lancedb[embeddings]` pulls
  `sentence-transformers` / OpenAI / Cohere / Gemini / Bedrock
  embedder shims so you can declare `LanceModel` rows that
  auto-embed on insert.

## 2. License

Apache-2.0 (file is `LICENSE` at the repo root). Whole codebase is
Apache-2.0 — no ELv2 / SSPL / BSL games at the storage layer or
the bindings.

## 3. Models supported

LanceDB is a **storage / retrieval** layer; the model axis is
**which embedder produces the vectors you store**. Bundled
embedding adapters (registered via `EmbeddingFunctionRegistry`):

- **OpenAI** — `text-embedding-3-small` / `-large` / `-ada-002`.
- **Cohere** — `embed-english-v3.0`, `embed-multilingual-v3.0`.
- **Gemini** — `text-embedding-004`, `gemini-embedding-001`.
- **Bedrock** — Titan, Cohere-on-Bedrock.
- **Sentence-Transformers** — any HF model (`all-MiniLM-L6-v2`,
  `bge-large-en-v1.5`, `e5-large-v2`, multilingual variants),
  fully offline.
- **Instructor** — instruction-tuned embeddings.
- **Jina, Voyage AI, watsonx, Ollama** — first-party adapters.
- **OpenCLIP / Imagebind** — multimodal (image + text + audio in
  the same vector space) for cross-modal retrieval — the use case
  the catalog page leads with.
- **Custom** — subclass `EmbeddingFunction`, register, ship.

Storage side accepts any vector dimensionality, sparse + dense in
the same row, plus arbitrary scalar columns (string / int / list /
struct / image bytes / timestamp) — everything is Arrow types.

## 4. MCP support

**No first-party MCP server.** LanceDB is a library, not a service;
no resource / tool / prompt surface to publish over MCP. Two
integration shapes work today:

- **Inside an MCP server you write** — your tool implementation
  opens the LanceDB directory and serves
  `db.open_table("docs").search(query_vec).limit(10)` as a
  retrieval tool. The whole tool is ~20 lines of Python.
- **As the vector store under a framework with MCP support** —
  [`haystack`](../haystack/), [`langflow`](../langflow/),
  [`mem0`](../mem0/), [`llmware`](../llmware/),
  [`scrapegraphai`](../scrapegraphai/) all support LanceDB as a
  vector backend; the framework's MCP server (when present) reads
  / writes through it transparently.

A community project `lancedb-mcp` exposes a generic vector-search
MCP surface over a LanceDB directory; not first-party, evaluate
before relying on it.

## 5. Sub-agent model

None — LanceDB is a substrate. Multi-agent / multi-process access
works because Lance datasets are versioned (each commit = a new
manifest pointing at append-only data files), so a writer and N
readers coexist via OS-level file locking + manifest versioning
without coordination overhead. Time-travel queries
(`db.open_table("docs", version=42)`) let an evaluator agent
reproduce exactly what a retriever agent saw at scoring time.

For composite retrieval pipelines, the Python `Reranker` interface
chains "vector top-K → reranker (Cohere / Cross-Encoder / Jina /
custom) → top-N" inside one `.search().rerank().limit()` call —
the retrieval-layer counterpart to a small ranking sub-agent.

## 6. Telemetry stance

**Off by default in OSS.** No analytics in the embedded library;
the database is a directory on your disk and only egresses when
you (a) call a remote embedder and (b) reach for the optional
managed cloud (`lancedb.connect("db://your-cloud-uri", api_key=...)`),
which is opt-in via API key.

The data lives in your filesystem (or your S3 / GCS / Azure Blob
bucket if you point `connect()` at an object-store URI) — treat
the directory like any other vector store and apply your normal
encryption / retention policy. LanceDB supports server-side
encryption when the storage backend (S3 / GCS) supports it.

For sensitive corpuses, pair with a local sentence-transformers
embedder + a local `lancedb.connect("./data")` and the entire
retrieval stack runs offline.

## 7. Prompt-cache strategy

**Not a prompt cache** — different layer. But the
**Lance columnar format gives you a scan cache for free**: queries
that touch the same columns mmap the same pages, so a workload of
"100 retrieval calls with the same filter" warms the OS page cache
and the second pass through is memory-bandwidth-bound, not
disk-bound. ANN indices (IVF_PQ, IVF_HNSW_PQ, IVF_HNSW_SQ, HNSW)
are loaded once per table-open and stay resident.

For "cache the LLM call that *uses* the retrieval result", layer
[`gptcache`](../gptcache/) on top of the LLM call site — orthogonal,
both cache hits compose multiplicatively to cut total latency.

## 8. Hot keybinds

No TUI, no CLI daemon. The Python surface is the everyday one:

```python
import lancedb
import pyarrow as pa
from lancedb.pydantic import LanceModel, Vector
from lancedb.embeddings import get_registry

# 1. Connect — local directory, S3, GCS, Azure, or managed cloud
db = lancedb.connect("./lancedb")

# 2. Declare a row schema with auto-embedding
embedder = get_registry().get("sentence-transformers").create(
    name="BAAI/bge-small-en-v1.5",
    device="cpu",
)
class Doc(LanceModel):
    text: str = embedder.SourceField()
    vector: Vector(embedder.ndims()) = embedder.VectorField()
    source: str
    chunk_id: int

# 3. Create + insert (vector is computed on insert, no manual embed call)
table = db.create_table("docs", schema=Doc, exist_ok=True)
table.add([{"text": "Pride and Prejudice was written by Jane Austen.",
            "source": "wiki", "chunk_id": 0}])

# 4. Query — vector + filter + reranker in one chain
results = (table.search("Who wrote Pride and Prejudice?")
                .where("source = 'wiki'")
                .limit(10)
                .to_pandas())

# 5. Hybrid (vector + BM25) — full-text index built once
table.create_fts_index("text", replace=True)
hybrid = (table.search("Pride and Prejudice", query_type="hybrid")
                .limit(5).to_pandas())

# 6. Versioned read — exactly what the retriever saw at eval time
old = db.open_table("docs", version=42)
```

Index types worth knowing:
`table.create_index(metric="cosine", index_type="IVF_HNSW_SQ",
                    num_partitions=256, num_sub_vectors=96)`
— IVF_HNSW_SQ is the everyday "good recall + small RAM + fast
build" pick for million- to ~hundred-million-row tables; HNSW for
small high-recall tables; IVF_PQ for billion-scale where RAM is
the constraint.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Embedded vector DB with the on-disk format
designed for the workload**. There is no server to run, no port to
secure, no Docker image to keep current — `pip install lancedb`
and `lancedb.connect("./data")` and you have a million-row
vector + full-text + hybrid retrieval store with sub-millisecond
ANN queries, mmap-friendly columnar storage, versioned commits
for time-travel reproducibility, and the same directory is
queryable from Rust, Node, Java, Go, Python, DuckDB, and Polars.
The Lance format itself was designed by ex-Cruise / ex-Eto folks
for the multimodal AI workload (image bytes + text + vector + label
in the same Arrow row), so storing 10 M images-with-embeddings is
literally one `table.add(df)` call where `df` is a Polars DataFrame
with a `bytes` column — no separate object store + index pointer
plumbing. Object-store-native: the same code runs on `./data`,
`s3://bucket/data`, `gs://bucket/data`, `az://container/data`.

**Weakness.** **Embedded means single-writer per table at the file
level**, so high-write-concurrency workloads (continuous ingestion
from N writers + thousands of readers) push you toward LanceDB
Cloud / a managed deployment rather than embedded mode. The Lance
v2 on-disk format is still adding minor capabilities across betas,
so the "pin the version" advice is real — a `lance` written by
v0.28 is read by v0.27 in most but not all cases. Reranker support
is solid for vector → rerank, but cross-encoder rerankers ship as
optional installs you wire yourself; the catalogue is narrower
than [`marqo`](../marqo/)'s server-side hybrid scoring or
Weaviate's reranker modules. Write-heavy workloads benefit from
periodic `table.optimize()` to compact small data files — easy to
forget, surfaces as slowly-degrading query latency.

**When to choose.**

- You want **a vector DB inside your process**, not behind a
  network hop — laptop, edge device, lambda, single-VM service,
  notebook, CI fixture.
- Your data is **multimodal** (images + text + structured columns)
  and you don't want to run an object store + a vector store + a
  metadata DB as three separate services with three failure modes.
- You need **time-travel reproducibility** — re-running an eval
  must use the exact retrieval index version that produced the
  original answers, and the framework should make that one call.
- You want **one storage format that Python, Rust, Node, Java,
  Go, DuckDB, and Polars all read** — same `./data` directory,
  no per-language bindings to keep in sync.
- You're prototyping at laptop scale and **want a clear graduation
  path** to S3-backed multi-reader (and managed cloud if needed)
  without rewriting your code.

**When not to choose.** You need **server-side embedding** (POST
text or image URL, get a vector back, no client-side embedder
service to run) — [`marqo`](../marqo/) is built around that
contract. You need **horizontal write scaling** with many parallel
writers and millisecond p99 across a cluster — Milvus / Qdrant /
Weaviate clusters are the production shape. You only need **a
side-table inside an existing Postgres** for moderate-scale RAG —
`pgvector` is simpler and lives in the DB you already operate. You
want **fully managed serverless** with no infra to think about —
Pinecone / Turbopuffer / Qdrant Cloud are closer to that shape.
