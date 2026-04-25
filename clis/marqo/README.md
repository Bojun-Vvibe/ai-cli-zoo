# marqo

> Snapshot date: 2026-04. Upstream: <https://github.com/marqo-ai/marqo>

"**End-to-end vector search engine for text and images.**" Marqo is a
self-contained vector search engine: one Docker image bundles an
inference layer (HuggingFace / OpenCLIP / sentence-transformers,
served on CPU / CUDA / Apple Silicon) **and** a vector store
(OpenSearch under the hood) **and** a query API. You add documents as
JSON, Marqo embeds them server-side and indexes them; you query with
text or an image URL, Marqo embeds the query server-side and returns
ranked hits — no separate embedding pipeline, no client-side model
loading.

## 1. Install footprint

- `docker run -p 8882:8882 marqoai/marqo:latest` is the install — one
  container, ports `8882` for the REST API. Apple Silicon image is
  `marqoai/marqo:latest-arm64`; CUDA via `--gpus all`.
- Python client: `pip install marqo` (TypeScript client also
  shipped). Both clients are thin wrappers over the REST surface.
- No standalone CLI binary; operator surface is `docker` + REST +
  language client. Self-hosted is the default; Marqo Cloud is the
  commercial managed option.
- Default ships with `hf/e5-base-v2` text embedder; swap models per
  index at create time (`open_clip/ViT-L-14/laion2b_s32b_b82k` for
  multimodal, `hf/bge-large-en-v1.5` for English-only retrieval, BYO
  HF checkpoint via `model_properties=...`).

## 2. Repo + version + license

- Repo: <https://github.com/marqo-ai/marqo>
- Latest release: **2.26.0** (2026-04-07)
- License: **Apache-2.0** —
  <https://github.com/marqo-ai/marqo/blob/mainline/LICENSE>
- Default branch: `mainline`
- Language: Python (with bundled OpenSearch / Vespa backends)

## 3. Models supported

Text: HuggingFace `sentence-transformers` family (`e5-*`, `bge-*`,
`all-MiniLM-L6-v2`, `all-mpnet-base-v2`, multilingual variants),
plus any HF causal/embedding checkpoint via `model_properties`.
Multimodal (text ↔ image, text ↔ image+text combo): OpenCLIP
(`ViT-B-32`, `ViT-L-14`, `ViT-H-14`, `ViT-bigG-14`) trained on
LAION-2B / LAION-5B, plus first-party Marqo trained models
(`Marqo/marqo-fashionCLIP`, `Marqo/marqo-fashionSigLIP`,
`Marqo/marqo-ecommerce-embeddings-B`,
`Marqo/marqo-ecommerce-embeddings-L`). Hybrid lexical + dense scoring
via `searchMethod="HYBRID"` (BM25 + dense fusion). The engine itself
does the inference — no external embedding API call per query.

## 4. When to use it

- You want **vector search without standing up an embedding pipeline
  separately** — Marqo embeds at index time and at query time
  inside the same container, so the wire format is "send text or image
  URL" rather than "send a 1024-dim float array".
- You want **multimodal search out of the box** — text-to-image,
  image-to-image, and image+text combo queries against an OpenCLIP
  index work without a separate vision tower service.
- You want a **production e-commerce / product-catalog search** — the
  Marqo team ships fine-tuned `marqo-fashionCLIP` /
  `marqo-ecommerce-embeddings` checkpoints that beat generic OpenCLIP
  on product-photo + title queries; `tensor` field weights and
  per-document score modifiers (price boost, recency decay, in-stock
  multiplier) are first-class on the search call.
- You want **hybrid lexical + dense retrieval as one query** —
  `searchMethod="HYBRID"` runs BM25 and dense in the same call and
  fuses the rankings server-side.

## 5. When NOT to use it

- You want a **lightweight vector library you embed in your Python
  process** — Marqo is a server with a 2 GB Docker image; pick
  `chromadb` (in-process), [`txtai`](../txtai/), or `faiss` for
  single-process workloads.
- You want a **managed serverless vector DB** with no ops — pick
  Pinecone / Weaviate Cloud / pgvector on RDS; Marqo Cloud exists but
  the value is in self-hosting the inference + storage together.
- You want **graph-RAG / knowledge graph retrieval** — Marqo is
  vector + lexical only; pick Neo4j / LanceDB-graph /
  [`txtai`](../txtai/)'s graph mode.
- You want **a coding agent or chat REPL** — Marqo is the retrieval
  substrate, not the LLM caller; pair with [`haystack`](../haystack/),
  [`langroid`](../langroid/), or any OpenAI-compatible client that
  fetches context from the Marqo REST API and feeds it to a chat
  model.

## 6. Closest alternatives

- **Weaviate** — vector DB with built-in modules for inference;
  closest peer in the "embed server-side" thesis, broader module
  catalog, AGPL/BSL licensing wrinkles vs. Marqo's Apache-2.0.
- **Qdrant** — pure vector DB (you bring embeddings); faster query
  layer in Rust, but you operate the embedder separately.
- **Milvus** — distributed vector DB; heavier ops surface, scales
  past Marqo's single-container deployment.
- [`txtai`](../txtai/) — portable single-process embeddings + index;
  in-process counterpart to Marqo's server-shaped deployment.
- **OpenSearch / Elasticsearch with k-NN plugin** — what Marqo runs
  internally; pick directly if you already operate an OpenSearch
  cluster and just want vector queries on top.

## 7. Repo health (snapshot)

- Active: 2.26.0 on 2026-04-07; release cadence ~3–4 weeks through
  the 2.x line; ~5k GitHub stars.
- Backed by Marqo (commercial parent shipping Marqo Cloud and
  fine-tuned e-commerce embedding models on HF); the OSS engine is
  the primary open surface and the same code that powers the cloud.
- Post-2.0 — the 2.x rewrite (mid-2024) consolidated the storage
  backend on Vespa and overhauled hybrid search; 1.x is deprecated.
  Index-create-time model selection plus per-search score modifiers
  have been stable through the 2.x line.
- Default branch is `mainline` (not `main`) — minor papercut for
  drive-by contributors.
