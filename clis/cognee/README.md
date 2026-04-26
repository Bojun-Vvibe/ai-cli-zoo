# cognee

> Snapshot date: 2026-04-26. Upstream: <https://github.com/topoteretes/cognee>

"**Knowledge Engine for AI Agent Memory in 6 lines of code.**"
Cognee is a memory / knowledge-graph layer for agents: ingest text,
docs, code, audio, or images via `cognee.add(...)`, run
`cognee.cognify()` to chunk, extract entities + relationships, embed,
and persist to a vector + graph + scalar triple-store, then query
with `cognee.search(...)` for graph-aware retrieval. The CLI is the
`cognee` Python package plus an MCP server (`cognee-mcp`) so any
MCP-aware client mounts the same memory surface as a tool.

## 1. Install footprint

- `pip install cognee` (core); per-backend extras `cognee[neo4j]`,
  `cognee[qdrant]`, `cognee[milvus]`, `cognee[postgres]`,
  `cognee[falkordb]`.
- MCP server: `uvx cognee-mcp` boots a stdio MCP server exposing
  `add` / `cognify` / `search` / `prune` / `list_data` tools.
- Defaults to fully local: SQLite (scalar) + LanceDB (vector) +
  NetworkX (graph) — zero external services for the smoke test.
- Production swap-ins: Postgres + Qdrant / Milvus / Weaviate +
  Neo4j / FalkorDB / Kuzu, each behind a uniform `DataPoint` API.

## 2. Repo + version + license

- Repo: <https://github.com/topoteretes/cognee>
- Latest release: **v1.0.4.dev0** (2026-04-25)
- License: **Apache-2.0** —
  <https://github.com/topoteretes/cognee/blob/main/LICENSE>
- Default branch: `main`
- Language: Python
- Stars: ~16.7k

## 3. Models supported

Provider-agnostic via LiteLLM for the extractor + reasoner LLM and
for the embedder: OpenAI, Anthropic, Gemini, Bedrock, Vertex,
Mistral, Cohere, Groq, DeepSeek, OpenRouter, Together, Fireworks,
plus any OpenAI-compatible local endpoint (Ollama / vLLM / LM Studio
/ llama.cpp). Embeddings cover OpenAI, Cohere, Voyage, FastEmbed,
HuggingFace sentence-transformers, and any OpenAI-compatible
embedding endpoint.

## 4. Notable angle

**Graph + vector + scalar as one queryable memory, with explicit
ontology hooks.** Where [`mem0`](../mem0/) and [`zep`](../zep/) lean
on temporal facts and episodic summaries, cognee's `cognify`
pipeline emits a typed entity-relationship graph (configurable
ontology, optional Pydantic-defined node types) alongside the
embedding index, so a query like "show every customer mentioned in
support tickets that also appear in the Q1 contract addenda" is one
graph traversal joined with vector recall, not three rounds of
re-ranking. The MCP server makes the same memory surface usable from
[`opencode`](../opencode/) / [`claude-code`](../claude-code/) /
[`crush`](../crush/) without writing any glue code.

## 5. Last verified

2026-04-26 via `gh api repos/topoteretes/cognee`.
