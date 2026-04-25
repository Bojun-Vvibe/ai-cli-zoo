# haystack

> Snapshot date: 2026-04. Upstream: <https://github.com/deepset-ai/haystack>

"**Open-source AI orchestration framework for building
context-engineered, production-ready LLM applications.**" Haystack is
a Python framework whose unit of composition is the **Pipeline** — a
typed DAG of `Component`s connected by socket names (`text` →
`documents` → `prompt` → `replies`). Components cover the full RAG /
agent surface: converters (PDF / DOCX / HTML / audio), splitters,
embedders, retrievers (BM25 + dense + hybrid + sparse), rankers,
prompt builders, generators (chat + tool-calling), routers, joiners,
and `Agent` (a `ToolInvoker` loop). The same `Pipeline.run({...})`
shape works in a notebook, as a `hayhooks` HTTP service, and inside a
Haystack Open Source MCP server.

## 1. Install footprint

- `pip install haystack-ai` (Python ≥ 3.9). Lean core; per-integration
  extras live in the separate `haystack-integrations` org and install
  by name (`pip install anthropic-haystack`,
  `pip install qdrant-haystack`, `pip install ollama-haystack`).
- `hayhooks` (separate package: `pip install hayhooks`) wraps any
  pipeline as a FastAPI endpoint with one YAML config; same pipeline
  also exposes itself as an MCP server via `hayhooks-mcp`.
- Library-first; not a single CLI binary. The `hayhooks` CLI subcommand
  surface (`hayhooks pipeline deploy`, `hayhooks run`) is the operator
  surface for serving.
- Pulls Pydantic v2, `lazy_imports`, no LangChain dependency.

## 2. Repo + version + license

- Repo: <https://github.com/deepset-ai/haystack>
- Latest release: **v2.28.0** (2026-04-20)
- License: **Apache-2.0** —
  <https://github.com/deepset-ai/haystack/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

OpenAI (Chat + Responses), Anthropic, Gemini (AI Studio + Vertex),
Bedrock (Anthropic / Titan / Llama / Mistral / Cohere), Azure OpenAI,
Cohere, Mistral, Groq, Together, Fireworks, DeepSeek, NVIDIA NIM,
SambaNova, Watsonx, HuggingFace (Inference + Local + TGI), Ollama,
vLLM, llama.cpp, plus any OpenAI-compatible endpoint via
`OpenAIChatGenerator(api_base_url=...)`. Embedders match the same
matrix; retrievers cover BM25 + dense + hybrid + sparse (SPLADE).
Vector store integrations: Qdrant, Weaviate, Pinecone, Chroma, Milvus,
OpenSearch, Elasticsearch, pgvector, MongoDB Atlas, AstraDB, Neo4j,
LanceDB, Marqo, plus an in-memory store for tests.

## 4. When to use it

- You want a **typed pipeline DAG with named sockets** as the
  composition contract — connect `splitter.documents` →
  `embedder.documents` → `writer.documents` and the framework
  validates the shape at `pipeline.connect(...)` time, not at runtime.
- You want **production RAG without LangChain underneath** — the
  retriever / ranker / prompt-builder / generator surface is
  Haystack-native and has been stable since the 2.0 rewrite (early
  2024); Pydantic v2 typing throughout.
- You want **the same pipeline to run as a Python call, an HTTP
  endpoint, and an MCP tool** without rewriting the orchestration —
  `hayhooks` wraps the pipeline as FastAPI, `hayhooks-mcp` republishes
  it on an MCP transport.
- You want **agent loops as a first-class component** — `Agent`
  wraps a `ToolInvoker` over typed `Tool` objects, plugs into any
  pipeline as just another component.

## 5. When NOT to use it

- You want a **terminal coding agent** that edits your repo with
  natural language — Haystack is the substrate for *building* one,
  not the agent itself; pick [`aider`](../aider/) /
  [`opencode`](../opencode/) / [`crush`](../crush/) /
  [`claude-code`](../claude-code/).
- You want **role-play multi-agent crews** (manager LLM auto-spawning
  workers) — Haystack composes typed components, not agent orgs;
  pick [`crewai`](../crewai/) / [`metagpt`](../metagpt/).
- You want a **prompt-compilation / few-shot search loop** — out of
  scope; pick [`dspy`](../dspy/).
- You want **a single-file install with no Python deps** — Haystack
  is library-shaped Python; pick a Go / Rust binary like
  [`aichat`](../aichat/) / [`mods`](../mods/) / [`fabric`](../fabric/).

## 6. Closest alternatives

- [`langroid`](../langroid/) — explicit `Task` graphs with typed
  message-passing; comparable "deterministic wiring" thesis without
  LangChain underneath.
- [`griptape`](../griptape/) — task-DAG framework with typed
  `Artifact`s and off-prompt Task Memory; similar production-RAG
  shape, smaller integration matrix.
- [`agno`](../agno/) — `Agent` / `Team` / `Workflow` primitives with
  hosted `AgentOS` runtime; richer agent surface, similar
  notebook → service path.
- [`txtai`](../txtai/) — single-binary embeddings + RAG with portable
  on-disk index files; thinner pipeline DSL but air-gap friendly out
  of the box.
- [`llmware`](../llmware/) — RAG stack betting on small specialised
  models; comparable end-to-end ingestion → vector → answer surface
  with a different model thesis.

## 7. Repo health (snapshot)

- Very active: v2.28.0 on 2026-04-20; release cadence ~3 weeks
  through the 2.x line; ~25k GitHub stars.
- Backed by deepset (commercial parent shipping deepset Cloud and
  deepset Studio); the OSS framework predates the cloud product and
  remains the primary open surface.
- Post-2.0 — the v2 rewrite (early 2024) replaced the v1 Pipeline /
  Node API; v1 is deprecated. `Component` / `Pipeline` / `Document` /
  `ChatMessage` surfaces have been stable since 2.0; integration
  packages move on their own cadence in the `haystack-integrations`
  org.
