# verba

> Snapshot date: 2026-04. Upstream: <https://github.com/weaviate/Verba>
> Pinned release: `v2.1.3` (HEAD `ca9a11c2f77c657c7416b5451825fa78322fc283`).
> License file: [`LICENSE`](https://github.com/weaviate/Verba/blob/main/LICENSE)
> (BSD-3-Clause).

`Verba` is Weaviate's batteries-included "Golden RAGtriever" — a
self-hostable RAG application that ships the full ingest →
chunk → embed → retrieve → generate stack as a FastAPI back-end
plus React front-end, with [`weaviate`](../weaviate/) as the
sole vector store and pluggable readers / chunkers / embedders /
generators behind a typed component interface. Drop in OpenAI,
Anthropic, Cohere, Ollama, or any HuggingFace model by editing
environment variables; ingest PDFs, GitHub repos, raw URLs,
Unstructured.io output, or `.txt` files via the web UI; query
through the chat interface or the FastAPI HTTP surface. The
catalog's reference for **"I have a Weaviate cluster and want a
production-shaped chat-over-docs UI on top in one Docker run"**,
sibling to the broader self-hostable RAG platforms
([`anything-llm`](../anything-llm/), [`docsgpt`](../docsgpt/),
[`khoj`](../khoj/)) — narrower because it's Weaviate-only,
sharper because every component is a swappable typed class.

## 1. Install footprint

- `pip install goldenverba` (Python 3.10–3.12) and `verba start`
  boots the FastAPI server on `:8000`. The React front-end is
  bundled as static assets — no separate Node build.
- `docker compose up -d` from the repo root brings up Verba +
  Weaviate + Ollama in one command (`docker-compose.yml` ships
  with sensible defaults; override via `.env`).
- Provider extras are runtime, not install-time — set
  `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `COHERE_API_KEY` /
  `OLLAMA_URL` and the corresponding components light up in the
  Settings panel.
- LLM-adjacent shapes: native readers for PDF, GitHub, Firecrawl,
  Unstructured.io, and a `.txt` / `.md` directory walker; native
  chunkers (token, sentence, semantic, recursive, code, HTML,
  Markdown); native embedders (OpenAI, Cohere, Ollama, SentenceTransformers,
  Weaviate's built-in `text2vec-*`).

## 2. Repo, version, license

- Repo: <https://github.com/weaviate/Verba>
- Latest release: `v2.1.3` (2025-07-14).
- HEAD pinned at this snapshot:
  `ca9a11c2f77c657c7416b5451825fa78322fc283`.
- License: BSD-3-Clause. License file at
  [`LICENSE`](https://github.com/weaviate/Verba/blob/main/LICENSE).
- Maintainer: Weaviate B.V. (the vector-store vendor).

## 3. What it actually does

Verba is structured as a typed component pipeline:

```
Reader  →  Chunker  →  Embedder  →  Retriever  →  Generator
(PDF /     (token /    (OpenAI /     (Weaviate    (OpenAI /
 GitHub /   sentence /  Cohere /      hybrid       Anthropic /
 URL /      semantic /  Ollama /      BM25 +       Cohere /
 Firecrawl) recursive)  HF)           vector)      Ollama)
```

Each stage is a Python class implementing a typed ABC; new
components register themselves on import and appear in the
front-end Settings panel without front-end code changes. A
typical session:

```bash
# Boot the stack
docker compose up -d

# Ingest a PDF via the web UI at http://localhost:8000
# or via the HTTP API:
curl -X POST http://localhost:8000/api/import \
  -F "files=@whitepaper.pdf" \
  -F "config={\"reader\":\"PDFReader\",\"chunker\":\"TokenChunker\",
              \"embedder\":\"OpenAIEmbedder\"}"

# Query
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"query":"summarise the abstract","generator":"OpenAIGenerator"}'
```

The same query returns through the Chat UI with inline source
citations clicking through to the original chunk.

## 4. MCP support

None first-party. Verba exposes a typed FastAPI surface
(`/api/query`, `/api/import`, `/api/get_documents`) — wrap with
[`fastmcp`](../fastmcp/) or any OpenAPI-to-MCP adapter to expose
the corpus as MCP tools to a host agent.

## 5. Sub-agent model

None — single retrieval + single generation per turn. The
relevant primitive is **typed component swap** — switching
embedder / generator / chunker is a Settings toggle, not a code
change. Multi-step or planner-shaped retrieval lives one tier
up ([`langgraph`](../langgraph/), [`crewai`](../crewai/),
[`deer-flow`](../deer-flow/)).

## 6. Telemetry stance

Off by default in self-host. Verba itself does not phone home;
the configured generator / embedder providers see prompts +
documents per their own posture. Pair with Ollama +
SentenceTransformers + a local Weaviate for a fully air-gapped
deployment.

## 7. When it is the right answer

- You already run Weaviate in production and want a real chat UI
  + ingest pipeline in front of it without authoring a React app.
- You want the component-swap shape (`Settings → Embedder →
  Cohere → save`) rather than a YAML config or a code change.
- BSD-3-Clause matters for your distribution path —
  [`anything-llm`](../anything-llm/) and [`docsgpt`](../docsgpt/)
  are MIT (broadly equivalent) but [`khoj`](../khoj/) is AGPL-3.0
  and [`firecrawl`](../firecrawl/) is AGPL-3.0; Verba is the
  permissive choice in this niche.

## 8. When to reach for something else

- You need a vector store other than Weaviate — Verba is
  hard-wired to Weaviate; pick [`anything-llm`](../anything-llm/)
  (per-workspace store selection across LanceDB / Chroma / Pinecone
  / Qdrant / Weaviate / Milvus / pgvector) or
  [`docsgpt`](../docsgpt/) (FAISS / Qdrant / Milvus / Mongo /
  Elastic / Pinecone) for store-agnosticism.
- You need an agent builder, MCP tool mounts, or third-party
  distribution surfaces (Chrome extension, Chatwoot bridge,
  Discord bot) — that's [`docsgpt`](../docsgpt/) /
  [`anything-llm`](../anything-llm/)'s lane.
- You only need the retrieval primitives without the UI — call
  Weaviate directly via the Python client; Verba is the UI +
  ingest layer, not new retrieval semantics.

## 9. Cross-references

- [`weaviate`](../weaviate/) — the vector store Verba is hard-wired
  to; the same hybrid BM25 + vector retrieval is what makes Verba's
  retrieval quality competitive without re-ranking.
- [`anything-llm`](../anything-llm/) — broader self-hostable
  RAG platform with pluggable vector stores.
- [`docsgpt`](../docsgpt/) — sibling RAG agent platform with a
  no-code agent builder and richer distribution surfaces.
- [`firecrawl`](../firecrawl/) — Verba's first-party URL reader
  delegates to Firecrawl for clean Markdown extraction.
