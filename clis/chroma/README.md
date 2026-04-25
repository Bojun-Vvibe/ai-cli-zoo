# chroma

> Snapshot date: 2026-04. Upstream: <https://github.com/chroma-core/chroma>
> License file: <https://github.com/chroma-core/chroma/blob/main/LICENSE>
> Pinned: `1.5.8` (2026-04-16, the unified Rust-core release line). The
> 1.x line is a Rust-rewritten core with the same Python / JS APIs as
> the legacy 0.x; pin the exact PyPI / npm version because on-disk
> format and HNSW persistence layout shifted between 0.5.x and 1.x.

A **batteries-included embedding database aimed at "make RAG work in
five lines"** — `chromadb.PersistentClient(path="./db")` opens a
local directory, `collection.add(documents=[...], ids=[...])` embeds
client-side via a default sentence-transformers model and persists,
`collection.query(query_texts=["..."], n_results=5)` returns ranked
matches with documents + metadata + distances in one call. The 1.x
core is rewritten in Rust (single binary, mmap-friendly, embedded
HNSW + SPANN options), but the everyday user-facing surface is
unchanged from the early Python-only era — the friction floor is
deliberately near zero.

## 1. Install footprint

- **Python embedded**: `pip install chromadb` (or `uv pip install
  chromadb`). ~150 MB after extras (the default embedder pulls
  `sentence-transformers` + `tokenizers` + `onnxruntime`). Python
  3.9–3.12.
- **Python client-only** (talk to a remote server, no local
  embedder): `pip install chromadb-client` — ~5 MB, no
  `sentence-transformers` dependency.
- **JS / TS**: `npm install chromadb` (server-talk) or `chromadb-default-embed`
  (bundles the default embedder via Transformers.js).
- **Server**: `chroma run --path ./chromadb_data --host 0.0.0.0
  --port 8000` (CLI subcommand of the Python install) or `docker
  run -p 8000:8000 -v ./chromadb_data:/data chromadb/chroma:1.5.8`.
- **Cloud**: Chroma Cloud is the managed offering (`chroma login`
  + `CloudClient(...)`) — opt-in, separate URL, separate billing.
- Workspace footprint: a `chromadb/` directory with `chroma.sqlite3`
  (metadata + embeddings) + a `*-uuid/` subdir per collection
  holding HNSW index files. Add to `.gitignore` unless committing
  a small reference index.
- The 1.x default storage is the new Rust-backed segment store —
  faster bulk ingest + lower memory than the 0.5.x line, same API.

## 2. License

Apache-2.0 (file is `LICENSE` at the repo root,
<https://github.com/chroma-core/chroma/blob/main/LICENSE>). Whole
codebase + clients are Apache-2.0 — no ELv2 / SSPL / BSL games.
Chroma Cloud is the commercial managed offering with no feature
gating on the OSS engine.

## 3. Models supported

Default embedder: `all-MiniLM-L6-v2` (sentence-transformers, 384-d,
~80 MB ONNX) — auto-downloaded on first `add()` and reused offline
forever after. The point of the default is "RAG works without you
choosing an embedder".

Bundled embedding-function adapters (`from chromadb.utils import
embedding_functions`):

- **OpenAI** — `OpenAIEmbeddingFunction(api_key=..., model_name=
  "text-embedding-3-small")`.
- **Cohere** — `embed-english-v3.0`, `embed-multilingual-v3.0`.
- **Google** — `text-embedding-004`, Gemini embeddings.
- **HuggingFace** — any `sentence-transformers` model via
  `SentenceTransformerEmbeddingFunction(model_name=...)` (offline
  after first download).
- **Instructor**, **Jina**, **Voyage AI**, **Mistral**, **Together**,
  **Ollama** (`OllamaEmbeddingFunction(url="http://localhost:11434",
  model_name="nomic-embed-text")`) — for fully offline embedding
  paired with [`ollama`](../ollama/) on the same box.
- **ONNX MiniLM** (the default, packaged) — works on CPU with no
  network.
- **Custom** — subclass `EmbeddingFunction[Documents]`, register on
  the collection, ship.

Storage side accepts arbitrary dimensionality, cosine / l2 / ip
distance, plus arbitrary metadata (string / int / float / bool, no
nested) for `where=` filtering at query time.

## 4. MCP support

**Official MCP server** (`chroma-mcp`, <https://github.com/chroma-
core/chroma-mcp>) — exposes collection / query / add tools so MCP-
aware agents ([`opencode`](../opencode/), [`claude-code`](../claude-code/),
[`crush`](../crush/), [`fast-agent`](../fast-agent/)) can read +
write semantic memory without touching the Python API. Run via
`uvx chroma-mcp --client-type persistent --data-dir ./db` (local
mode) or `--client-type http --host ... --port ...` (point at a
running server).

The base server also exposes a stable REST API (`POST
/api/v2/tenants/{t}/databases/{d}/collections/{c}/query`), so any
"MCP-from-OpenAPI" bridge gives you a tool surface over the full
collections / query / add API in one command.

## 5. Sub-agent model

None — Chroma is a substrate. Multi-process readers + a single
writer per database coexist via SQLite WAL + segment locking,
which is the same shape as [`lancedb`](../lancedb/). Distinct from
LanceDB: there is no time-travel / version-pinning surface — a
query reflects the latest commit, full stop. For evaluator
reproducibility you snapshot the directory (rsync / S3 sync) and
point a separate `PersistentClient` at the snapshot path.

The server mode (`chroma run`) handles N writer + N reader
concurrency via a single Rust process; horizontal sharding lives
in Chroma Cloud, not the OSS engine.

For composite retrieval pipelines, query takes `where=` (metadata
predicate), `where_document=` (substring filter on the document
text), and `include=["documents","metadatas","distances","embeddings"]`
in one call — the engine does the filter-then-ANN combination, you
get one list back. Reranking is your problem (run a cross-encoder
on the top-K returned).

## 6. Telemetry stance

**Anonymous product telemetry is on by default** in the OSS engine
— `chromadb` POSTs anonymised event counts (collection_add,
collection_query, etc.) to PostHog. Disable globally by setting
`anonymized_telemetry=False` in the `Settings(...)` you pass to
the client constructor, or by exporting `ANONYMIZED_TELEMETRY=False`
before first import. The opt-out is one line and is documented in
the README.

Data plane has no egress: documents + embeddings + metadata live on
the disk volume you mount (or in `:memory:` for `EphemeralClient`),
queries terminate in the engine, and the optional Chroma Cloud is a
separate hosted product reached via a different URL + API key.

For sensitive corpuses, set `chroma_server_authn_provider` +
`chroma_server_authn_credentials` to require a bearer token on
every server request, run behind your own TLS, and pair with a
local embedder (Sentence-Transformers ONNX or
[`ollama`](../ollama/) + `nomic-embed-text`) so no document text
ever leaves the box.

## 7. Prompt-cache strategy

**Not a prompt cache** — different layer. The 1.x Rust core's
segment store gives you a "warm working set" effect for free: hot
HNSW segments stay in the OS page cache, so a workload of "10 K
queries with overlapping ANN neighbourhoods" steady-states in
single-digit milliseconds on commodity NVMe.

For "cache the LLM call that *uses* the retrieval result", layer
[`gptcache`](../gptcache/) on top of the LLM call site — orthogonal,
both cache hits compose multiplicatively to cut total latency.

## 8. Hot keybinds

No TUI. The everyday surface is the Python client; for inspection
use the JS playground at the server `/api/v2/heartbeat` +
collection endpoints, or `chromadb-admin` (community).

```python
import chromadb
from chromadb.utils import embedding_functions

# 1. Connect — local persistent directory
client = chromadb.PersistentClient(path="./chromadb")

# 2. (Optional) pick an embedder — default is all-MiniLM-L6-v2 ONNX
embedder = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="BAAI/bge-small-en-v1.5",
)

# 3. Get-or-create a collection with metadata + embedding policy
collection = client.get_or_create_collection(
    name="docs",
    embedding_function=embedder,
    metadata={"hnsw:space": "cosine", "hnsw:M": 32, "hnsw:construction_ef": 200},
)

# 4. Add — embeddings computed on insert if you pass `documents=`
collection.add(
    documents=[
        "Pride and Prejudice was written by Jane Austen.",
        "Moby Dick was written by Herman Melville.",
    ],
    metadatas=[{"source": "wiki", "year": 1813}, {"source": "wiki", "year": 1851}],
    ids=["doc-1", "doc-2"],
)

# 5. Query — vector + metadata filter + document substring filter in one call
results = collection.query(
    query_texts=["Who wrote Pride and Prejudice?"],
    n_results=5,
    where={"source": "wiki"},
    where_document={"$contains": "Austen"},
    include=["documents", "metadatas", "distances"],
)

# 6. Update / upsert / delete — same shape
collection.upsert(ids=["doc-1"], documents=["..."], metadatas=[{"source": "wiki", "year": 1813}])
collection.delete(where={"year": {"$lt": 1800}})
```

HNSW knobs worth knowing (passed via collection metadata at
creation time):
- `hnsw:M` — graph degree (default 16; 32–48 for high-dim ≥1024-d).
- `hnsw:construction_ef` — build-time exploration (default 100;
  raise for one-time index builds, lower for streaming ingest).
- `hnsw:search_ef` — query-time exploration (default 10; raise per-
  query for higher recall at higher latency).
- `hnsw:space` — `cosine` / `l2` / `ip`.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **The shortest distance from "I have a list of
documents" to "I have working RAG"**. `pip install chromadb` →
`PersistentClient("./db")` → `collection.add(documents=[...],
ids=[...])` → `collection.query(query_texts=["..."], n_results=5)`
is four lines and the default sentence-transformers embedder is
auto-downloaded on first add, so there is no embedder configuration
to think about until you outgrow it. The Python and JS APIs are
identical in shape, the same code works in `EphemeralClient`
(in-memory tests), `PersistentClient` (laptop dev), `HttpClient`
(team-shared server), and `CloudClient` (managed) — the prototype-
to-production path is one constructor change, not a code rewrite.

**Weakness.** **No time-travel / version pinning** in the OSS
engine — a query reflects the latest commit, period; evaluator
reproducibility means snapshotting the directory on the side, not
`db.open_table("docs", version=42)` like
[`lancedb`](../lancedb/) gives you. **Telemetry on by default**
(opt-out is one line, but it is on) — surprising for users who
expect "embedded DB == zero egress". The `where_document` filter
is substring-only (no regex, no full-text scoring) — for real
hybrid (BM25 + dense) retrieval reach for [`qdrant`](../qdrant/) or
[`marqo`](../marqo/) instead. The 1.x rewrite changed on-disk
format from 0.5.x — pin the version and migrate on a planned
window, not by accident.

**When to choose.**

- You want **the lowest-friction "library + default embedder"
  combination** — the four-line snippet above is the pitch.
- You want **the same API across embedded → server → managed cloud**
  with no rewrite — `EphemeralClient` / `PersistentClient` /
  `HttpClient` / `CloudClient` swap by constructor name only.
- You want **first-class metadata filtering at query time** that
  composes with vector search in one round-trip.
- You're prototyping at laptop scale and want **a stable API surface
  that has not churned** since the early days, while the underlying
  storage engine quietly got rewritten in Rust under you.
- You want **Apache-2.0 with no enterprise carveout** in the OSS
  engine.

**When not to choose.** You need **time-travel reproducibility**
(re-run an eval against the exact retrieval state) — pick
[`lancedb`](../lancedb/) for `version=42` reads. You need **server-
side embedding** (POST text or image URL, never ship a vector) —
pick [`marqo`](../marqo/). You need **production-shaped hybrid
retrieval** with sparse-vector indices, payload-filter quantisation,
and Raft cluster mode — pick [`qdrant`](../qdrant/). You only need
a **side-table inside an existing Postgres** — `pgvector` is
simpler and lives in the DB you already operate. You want
**telemetry off by default with no setting to flip** — pick
[`lancedb`](../lancedb/) or [`qdrant`](../qdrant/) and skip the
opt-out step.
