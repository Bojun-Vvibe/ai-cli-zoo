# upsonic

> Snapshot date: 2026-04. Upstream: <https://github.com/Upsonic/Upsonic>

"**Build autonomous AI agents in Python.**" Upsonic is a Python agent
framework focused on **reliability** — every `Agent` is wrapped in a
configurable verification layer (input safety, task validation, output
validation, hallucination check, agent-call retries) so production
deployments fail loud rather than silently degrading. Primitives:
`Agent`, `Task`, `Team`, `Knowledge` (RAG), `Memory`, `Canvas` (shared
scratchpad), and a tool catalogue covering shell, browser, file IO,
HTTP, search, and 100+ MCP servers. The CLI surface is `upsonic`
(server + project scaffolding) plus `pip install upsonic` for
in-process use.

## 1. Install footprint

- `pip install upsonic` (core); per-feature extras `upsonic[browser]`,
  `upsonic[graph]`, `upsonic[storage-redis]`, `upsonic[loaders-pdf]`,
  `upsonic[search-tavily]`, etc.
- Or `docker run -p 7541:7541 upsonic/server:latest` to run the
  optional gRPC + REST server stand-alone.
- `upsonic` CLI installed alongside (`upsonic server start`,
  `upsonic init`, `upsonic doctor`).
- Python ≥ 3.10. Pulls Pydantic v2, `httpx`, OpenAI / Anthropic client
  libs lazily.
- 100+ tool integrations including shell, Python REPL, file IO,
  Playwright browser, vector stores (Chroma / Qdrant / Pinecone /
  Weaviate / pgvector / Milvus / FAISS / LanceDB), SQL (Postgres /
  SQLite / MySQL / DuckDB), search (Tavily / Exa / DuckDuckGo / SerpAPI
  / Brave), MCP (any stdio or HTTP server), Slack, Notion, GitHub,
  Linear.

## 2. Repo + version + license

- Repo: <https://github.com/Upsonic/Upsonic>
- Latest release: **v0.76.2** (2026-04-22)
- License: **MIT** —
  <https://github.com/Upsonic/Upsonic/blob/master/LICENCE>
- Default branch: `master`
- Language: Python

## 3. Models supported

Anthropic (Claude), OpenAI (Chat + Responses), Google (Gemini AI Studio
+ Vertex), AWS Bedrock, Azure OpenAI, Mistral, Cohere, Groq, DeepSeek,
xAI Grok, Together, Fireworks, Cerebras, OpenRouter, Perplexity,
Hugging Face Inference, Ollama (any local GGUF), LM Studio, vLLM,
llama.cpp, any OpenAI-compatible endpoint via the
`OpenAIChat(base_url=...)` constructor. Routing is delegated to
[`pydantic-ai`](../pydantic-ai/) under the hood, so anything its model
abstraction supports works in Upsonic. Per-`Agent` model assignment is
first-class.

## 4. MCP support

Yes (client). `from upsonic.tools import MCPTool` mounts an MCP
server's tool surface (stdio or HTTP) onto any agent. Multiple MCP
servers per agent are supported and Upsonic handles tool-name
namespacing automatically. No first-party `upsonic --mcp-server`
mode (the runtime is HTTP / gRPC, not stdio MCP), but tools you write
in Upsonic can be re-exposed as an MCP server via the standard
`fastmcp` adapter.

## 5. Sub-agent model

Three composition primitives:

- `Agent` — the unit. Has `model`, `tools`, `memory`, `knowledge`
  (RAG), `system_prompt`, optional `reliability_layer`
  (`ValidatorAgent`, `EditorAgent`, `RotatorAgent`).
- `Task` — a typed unit of work with `description`, `response_format`
  (Pydantic model), `context`, optional `tools`, and optional
  `attachments`. Tasks are passed to `Agent.do(task)` or
  `Team.do(task)`.
- `Team` — multiple `Agent`s coordinated either round-robin
  (`round_robin=True`) or via a manager-LLM that picks the best
  responder per task. Teams share a `Canvas` (a typed shared scratchpad
  used to pass intermediate state between agents without trampling
  each other's memory).

The reliability layer is the differentiator: each `Agent` can be
wrapped with up to five auxiliary agents that validate inputs, retry
tool calls, edit outputs, and detect hallucinations against the
provided `knowledge` — these run as sub-agents in the same Python
process and share the parent's trace.

## 6. Telemetry stance

Off by default in OSS. Egress is your model providers + your DB +
optional MCP servers. Optional opt-in tracing via **Upsonic Cloud**
(hosted observability + agent registry) which streams spans from your
local process — turn off by not setting `UPSONIC_API_KEY`.
OpenTelemetry export is opt-in via the standard OTel env vars; spans
are emitted for every `Agent.do`, tool call, validator pass, and
retry. The optional standalone server (`docker run upsonic/server`)
ships with no outbound calls beyond your configured providers.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Reliability is a first-class wrapper, not an
afterthought.** Where most Python agent frameworks stop at "the agent
called your tool, here is the response, good luck", Upsonic ships a
configurable `reliability_layer` you bolt onto any `Agent`:
`ValidatorAgent` checks the task description against the proposed
plan, `EditorAgent` rewrites outputs that violate the response schema,
a hallucination check cross-references claims against your `Knowledge`
RAG, and tool-call retries are wrapped in their own audit. You declare
the schema (`response_format=MyPydanticModel`) and the framework keeps
retrying + repairing until it parses. The shared `Canvas` between
team members lets multiple agents collaborate on the same artefact
(a markdown doc, a JSON proposal, a PR description) without each one
losing the others' edits — much closer to how humans actually share
state than passing strings between LLM calls.

**Weakness.** **Framework-shaped, not CLI-shaped, and the reliability
layer costs tokens.** The `upsonic` binary is a server launcher + a
project scaffolder; the day-to-day surface is `import upsonic` in a
Python file. There is no "edit my repo with natural language" mode out
of the box. The validator / editor / hallucination-check agents add
2–4× the token cost of an unwrapped agent call — worth it for
production reliability, expensive for a quick prototype (turn the
layer off via `reliability_layer=None` while iterating). Rapid release
cadence (v0.7x in 2026-04, weekly minors with breaking changes); pin a
version in production. Documentation lags the API surface in places —
the cookbook + GitHub examples are the source of truth.

**When to choose.** You are **building** a Python agentic product
where **incorrect output is worse than no output** (financial
extraction, medical summarisation, legal-doc QA, customer-support
ticket routing) and want the framework + the validator stack + typed
response schemas + per-agent retries + a shared collaboration scratchpad
all from one library. Pair with [`pydantic-ai`](../pydantic-ai/) if you
want to drop down to its raw API (Upsonic uses it under the hood), with
[`agno`](../agno/) if you prefer the production-runtime focus over the
reliability focus, or with [`instructor`](../instructor/) for the same
typed-output discipline at the LLM-call layer instead of the agent
layer. Skip if you want a terminal coding agent (use
[`aider`](../aider/) / [`opencode`](../opencode/) / [`crush`](../crush/)
instead), a TypeScript stack (use [`voltagent`](../voltagent/) /
[`mastra`](../mastra/)), or a minimal "just call the LLM" Unix filter
(use [`mods`](../mods/) / [`llm`](../llm/) / [`smartcat`](../smartcat/)).
