# langflow

> Snapshot date: 2026-04. Upstream: <https://github.com/langflow-ai/langflow>

"**A powerful tool for building and deploying AI-powered agents and
workflows.**" Langflow is a visual flow editor (React canvas) backed
by a Python runtime: drag typed `Component` nodes onto a canvas
(LLM, prompt, agent, tool, loader, splitter, embedder, vector store,
output parser, conditional, loop), wire their inputs/outputs, and the
same flow runs as a chat playground, an HTTP endpoint
(`POST /api/v1/run/<flow-id>`), or an MCP server. Each flow is JSON
on disk so designers and engineers diff and version-control the same
artifact.

## 1. Install footprint

- `uv pip install langflow` (Python ≥ 3.10, < 3.13). Then
  `langflow run` boots the React UI + FastAPI backend at
  `http://localhost:7860`.
- Docker: `docker run -p 7860:7860 langflowai/langflow:latest` — one
  container, no external DB required for the SQLite default.
- Bundled component library covers ~200 nodes out of the box:
  generators (OpenAI / Anthropic / Gemini / Bedrock / Azure / Groq /
  Mistral / Cohere / DeepSeek / Together / Fireworks / OpenRouter /
  Ollama / vLLM / LM Studio / llama.cpp), embedders, vector stores
  (Astra DB / Pinecone / Chroma / Qdrant / Weaviate / Milvus /
  OpenSearch / pgvector / Mongo / Redis / Couchbase / SupaBase),
  document loaders (PDF / DOCX / HTML / Notion / Confluence / GitHub /
  Google Drive / S3), agents, tools (Python REPL, web search,
  Composio, MCP client).
- Auth via per-provider env vars or in-canvas `SecretStr` inputs;
  flows export to JSON or pure Python script (`langflow export`).

## 2. Repo + version + license

- Repo: <https://github.com/langflow-ai/langflow>
- Latest release: **v1.9.1** (2026-04-24)
- License: **MIT** —
  <https://github.com/langflow-ai/langflow/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (frontend in TypeScript / React)

## 3. Models supported

OpenAI (Chat + Responses), Anthropic, Gemini (AI Studio + Vertex),
Bedrock (Anthropic / Titan / Llama / Mistral / Cohere), Azure OpenAI,
Groq, Mistral, Cohere, DeepSeek, Together, Fireworks, OpenRouter,
NVIDIA NIM, Perplexity, xAI Grok, Sambanova, plus local: Ollama, vLLM,
LM Studio, llama.cpp, any OpenAI-compatible endpoint via the
`OpenAIModel` component's `base_url` input. Per-component model
assignment is first-class — different LLM nodes in the same flow can
point at different providers (cheap model for routing, smart model
for synthesis).

## 4. When to use it

- You want **a visual canvas as the authoring surface** for an agent /
  RAG pipeline — drag-drop is the workflow, the JSON is the
  artifact, and "send the screenshot of the flow to a teammate"
  doubles as documentation.
- You want **prompt + tool + retrieval + LLM wiring you can hand off
  to non-Python users** — PMs and prompt engineers iterate in the
  canvas, engineers consume the same JSON via the runtime API or
  export it to a `.py` script for production.
- You want the **same flow available as chat, HTTP endpoint, and MCP
  server** — `langflow run` ships the playground, `POST /api/v1/run`
  is the HTTP surface, and the built-in MCP server publishes flows as
  callable tools to MCP-aware agents ([opencode](../opencode/) /
  [claude-code](../claude-code/) / [crush](../crush/) /
  [fast-agent](../fast-agent/)).
- You want a **batteries-included component library** — the ~200
  bundled nodes cover most "load PDFs, embed, store, retrieve, prompt,
  call tool, return JSON" workloads without writing component code.

## 5. When NOT to use it

- You want a **terminal-first authoring loop** — Langflow's value
  *is* the visual canvas; if you live in `vim` and want pipeline
  code in `git`, pick [`haystack`](../haystack/) /
  [`langroid`](../langroid/) / [`griptape`](../griptape/) /
  [`fast-agent`](../fast-agent/).
- You want a **lightweight library you import in your service** —
  Langflow is a Python server + React frontend; the runtime is heavy
  for headless use (uvicorn + sqlite + a websocket layer) compared
  to a plain `pip install` framework.
- You want **typed composition with mypy / pyright catching wiring
  errors** — node sockets are validated at runtime in the canvas;
  pick [`pydantic-ai`](../pydantic-ai/) / [`mirascope`](../mirascope/)
  for type-system-first wiring.
- You want a **single binary / no-Python install** — Langflow is
  Python + React; pick a Go binary like [`crush`](../crush/) for
  CLI-shaped agent work.

## 6. Closest alternatives

- **Flowise** — TypeScript visual flow editor with similar drag-drop
  surface; closest peer, different stack (Node vs. Python).
- [`fast-agent`](../fast-agent/) — code-first MCP-native agent crews
  in ~30 lines of Python; the "live in code" counterpart.
- [`haystack`](../haystack/) — typed pipeline DAG without a visual
  canvas; same composition thesis, code as the authoring surface.
- [`agno`](../agno/) — `Agent` / `Team` / `Workflow` primitives with
  hosted `AgentOS` runtime; library-shaped vs. Langflow's
  canvas-shaped.
- [`db-gpt`](../db-gpt/) — AWEL DAG of typed operators with Python as
  the authoring surface; data-engineering-flavoured workflow vs.
  Langflow's drag-drop generality.

## 7. Repo health (snapshot)

- Extremely active: v1.9.1 on 2026-04-24; release cadence ~1–2 weeks
  through the 1.x line; ~147k GitHub stars (one of the most-starred
  AI projects on GitHub).
- Backed by Langflow Inc. (acquired by DataStax in 2025); the OSS
  project is the primary open surface and continues independent of
  the DataStax-managed offering.
- Post-1.0 — the 1.x line (since early 2024) overhauled the component
  model and the React frontend; the 0.x flow JSON format is no longer
  loadable. `Component` socket types have been stable through 1.x,
  but bundled component package paths churn occasionally — pin the
  Langflow version in production.
- Component contributions (PRs adding new bundled nodes for a
  provider or vector store) are the dominant changelog item;
  framework-internal changes are rarer post-1.0.
