# chonkie

> Snapshot date: 2026-04-26. Upstream: <https://github.com/chonkie-inc/chonkie>

"**CHONK docs with Chonkie — the lightweight ingestion library
for fast, efficient and robust RAG pipelines.**" Chonkie is the
chunking-first half of a RAG stack: a single Python (and
TypeScript port) library that gives you nine purpose-built
chunkers (token, sentence, recursive, semantic, SDPM, late, code,
neural, slumber) plus the optional plumbing to embed, refine, and
push the resulting chunks into a vector store, behind a CLI and
Python API that stays under one heavy install.

## 1. Install footprint

- Library: `pip install chonkie` for the lean default (token +
  sentence + recursive + code chunkers, no model deps); extras
  like `chonkie[semantic]`, `chonkie[neural]`, `chonkie[openai]`,
  `chonkie[cohere]`, `chonkie[qdrant]`, `chonkie[chroma]`,
  `chonkie[all]` opt into model-backed chunkers, embedding
  providers, refineries, and vector-store handshakes.
- CLI: `chonkie` entrypoint for one-shot chunking of files /
  directories with a chosen chunker + tokenizer; pipes JSONL of
  chunks to stdout for downstream tools.
- TypeScript port: `chonkie-ts` is published under the same org
  for Node / edge runtimes that need the same chunker semantics.

## 2. Repo + version + license

- Repo: <https://github.com/chonkie-inc/chonkie>
- Latest release: **v1.6.4** (published 2026-04-21)
- License: **MIT** —
  <https://github.com/chonkie-inc/chonkie/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (TS sibling repo `chonkie-inc/chonkie-ts`)
- Stars: ~3.9k

## 3. Models supported

- Tokenizers: `tiktoken` (default, cached), Hugging Face fast
  tokenizers, `tokenizers` Rust bindings, callable token-counters
  for custom byte-pair / sentencepiece schemes.
- Embeddings (for `SemanticChunker`, `SDPMChunker`,
  `LateChunker`, `EmbeddingsRefinery`): sentence-transformers
  (local), OpenAI, Cohere, Jina, VoyageAI, Model2Vec, plus a
  `CustomEmbeddings` shim for any callable.
- Neural chunker uses a small fine-tuned encoder
  (`mirth/chonky_*`) that runs CPU-fast for boundary detection
  without an LLM call.
- Vector-store handshakes: Qdrant, Chroma, Pinecone, Weaviate,
  Turbopuffer, pgvector — chunks go straight from chunker →
  refinery → handshake without manual `upsert` glue.

## 4. Notable angle

**Chunking as a first-class, swappable stage instead of an
afterthought baked into a framework.** Where
[`llama-index`](../llama-index/) and
[`haystack`](../haystack/) ship one or two chunkers buried inside
a larger pipeline and where ingestion-as-a-service projects like
[`docling`](../docling/) or [`marker`](../marker/) focus on
*producing* clean text, Chonkie's product is the chunker zoo
itself: pick `RecursiveChunker` for code-aware boundaries,
`SemanticChunker` for embedding-similarity cuts, `SDPMChunker`
(Semantic Double-Pass Merging) for tight topical windows,
`LateChunker` to embed the whole doc once and slice the embedding
afterwards, `NeuralChunker` for learned boundaries, `CodeChunker`
for AST-aware language-specific cuts. Compose any of them with a
`Refinery` (overlap, embeddings, summary) and one of the vector-
store `Handshake` adapters and you get an end-to-end ingest
pipeline whose only opinion is that chunks are the unit of work.
The lean default install is small enough to drop into a serverless
function; the `[all]` extra is heavy but still narrower than a
full RAG framework.

## 5. Last verified

2026-04-26 via `gh api repos/chonkie-inc/chonkie` and
`gh api repos/chonkie-inc/chonkie/releases/latest` → v1.6.4.
