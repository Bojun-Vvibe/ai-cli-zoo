# agno

> Snapshot date: 2026-04. Upstream: <https://github.com/agno-agi/agno>

"**Build, run, and manage agentic software at scale.**" Agno is a
Python agent framework + serving runtime: declare an `Agent` (model,
tools, memory, knowledge, guardrails), compose `Team`s and
`Workflow`s, then expose them as a stateless FastAPI service via
`AgentOS`. The CLI surface is `agno` (operate the local AgentOS
instance, scaffold projects, run cookbook recipes) plus a `uvx
... fastapi dev agno_assist.py` quickstart that boots a production
API in under twenty lines of Python.

## 1. Install footprint

- `pip install -U agno` (core); `pip install "agno[os]"` adds the
  FastAPI / `AgentOS` runtime; per-provider extras `agno[anthropic]`,
  `agno[openai]`, `agno[google]`, etc.
- `agno` CLI installed alongside (`agno --help`, `agno setup`,
  `agno ws create`, `agno ws up`).
- Python ≥ 3.9. Pulls Pydantic v2, `httpx`, FastAPI / Uvicorn (with
  `[os]`).
- 100+ first-party tool integrations: shell, Python REPL, file IO,
  HTTP, SQL (Postgres / SQLite / DuckDB / Snowflake), vector stores
  (pgvector / LanceDB / Pinecone / Qdrant / Weaviate / Chroma / Milvus
  / Cassandra), MCP, AWS / GCP, Slack, Notion, Linear, Jira, GitHub,
  GitLab, browser (Playwright), code execution (E2B, Modal,
  Daytona), search (DuckDuckGo / Tavily / Exa / Brave / SerpAPI / Bing).

## 2. Repo + version + license

- Repo: <https://github.com/agno-agi/agno>
- Latest release: **v2.6.1** (2026-04-24)
- License: **Apache-2.0** —
  <https://github.com/agno-agi/agno/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Anthropic (Claude), OpenAI (Chat + Responses), Google (Gemini AI
Studio + Vertex), AWS Bedrock, Azure OpenAI, Mistral, Cohere, Groq,
DeepSeek, xAI Grok, Together, Fireworks, Cerebras, SambaNova, Nvidia
NIM, Hugging Face Inference, OpenRouter, Perplexity, IBM watsonx,
Ollama (any local GGUF), LiteLLM-routed (any of ~100 providers),
LM Studio, vLLM, llama.cpp, any OpenAI-compatible endpoint via the
`OpenAIChat(base_url=...)` constructor. Per-agent model assignment is
first-class: each `Agent`, `Team` member, and `Workflow` step can use
a different model + provider.

## 4. MCP support

Yes (client). `agno.tools.mcp.MCPTools(url=...)` mounts an MCP
server's tool surface onto any agent. Multiple MCP servers per
agent supported; tool routing is automatic. No first-party
`agno --mcp-server` mode (the runtime is FastAPI / HTTP, not stdio
MCP), but the OS server can be wrapped externally.

## 5. Sub-agent model

Three composition primitives:

- `Agent` — the unit. Has `model`, `tools`, `db` (memory), `knowledge`
  (RAG), `instructions`, optional `guardrails`.
- `Team` — manager LLM coordinates worker agents. Modes: `route`
  (manager picks one worker), `coordinate` (manager plans across
  workers), `collaborate` (parallel + synthesis). Each worker keeps
  its own model + tools + memory.
- `Workflow` — explicit Python-defined DAG of steps; each step can be
  an `Agent`, a `Team`, a function, or a nested `Workflow`. Branching,
  looping, and conditional edges are first-class.

Sessions and memory are scoped per-user / per-conversation by the
runtime, so a `Team` of N agents shares a thread without trampling
state.

## 6. Telemetry stance

Off by default in OSS. The `AgentOS` runtime is fully self-hostable;
egress is your model providers + your DB. Optional opt-in tracing via
the **AgentOS UI** (hosted at `os.agno.com`) which connects to your
local FastAPI process for monitoring / replay — turn off by not
calling `tracing=True` and not connecting the UI. OpenTelemetry export
is opt-in via the standard OTel env vars.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **The agent you build *is* the production API,
in twenty lines of Python.** Where most multi-agent frameworks stop
at "you have an `Agent` object" and leave deployment to you, Agno
ships `AgentOS` — a stateless, session-scoped FastAPI app that wraps
your `Agent` / `Team` / `Workflow` graph, handles per-user memory in
the configured `db`, streams responses, exposes a stable HTTP +
WebSocket surface, and integrates with the AgentOS UI for
observability without you writing a controller. Per-agent model
assignment in the constructor (`model=Claude(...)` here,
`model=GPT5(...)` there, `model=Ollama(...)` for the cheap reranker)
makes provider-mixing the default. The same Python file runs as a
notebook experiment, a `uvx ... fastapi dev` dev server, and a
production deployment behind your reverse proxy — the surface does
not change.

**Weakness.** **Framework-shaped, not CLI-shaped.** The `agno` binary
is mostly a workspace scaffolder + `agno ws up` Docker-Compose
launcher; the day-to-day surface is `import agno` in a Python file
and `uvicorn` / `fastapi`. There is no "edit my repo with natural
language" mode out of the box — Agno is the substrate for *building*
that, not a turnkey coding agent like [`aider`](../aider/) /
[`claude-code`](../claude-code/) / [`opencode`](../opencode/). Rapid
2.x release cadence (v2.0 → v2.6 in months) means breaking API
changes between minor versions; pin a version in production. The
hosted AgentOS UI is the best operational surface but is a SaaS
endpoint — air-gapped users get the framework + their own dashboards
only.

**When to choose.** You are **building** an agentic application
(customer support bot, internal RAG assistant, multi-step automation,
trading agent, research crew) and want the framework + the runtime +
per-user memory + tracing + hosted observability all from one library,
with provider-mixing as the default and a real path from notebook to
production HTTP service. Pair with [`kit`](../kit/) to package the
agent + its model + its prompts as a versioned ModelKit artifact, or
with [`pydantic-ai`](../pydantic-ai/) if you prefer the type-system-
first API shape over Agno's tool-rich batteries-included one. Skip if
you want a terminal coding agent (use [`aider`](../aider/) /
[`opencode`](../opencode/) / [`crush`](../crush/) instead) or a
single-prompt Unix filter (use [`mods`](../mods/) /
[`smartcat`](../smartcat/) / [`llm`](../llm/)).
