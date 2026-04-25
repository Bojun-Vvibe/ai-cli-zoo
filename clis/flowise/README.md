# flowise

> Snapshot date: 2026-04. Upstream: <https://github.com/FlowiseAI/Flowise>

"**Drag-and-drop builder for LLM apps and agents — Node.js
counterpart to [`langflow`](../langflow/).**" `flowise` is a
visual canvas (React Flow on the front, Express + TypeORM on the
back) where chains, agents, retrievers, vector stores, document
loaders, embedders, models, and tools are nodes you wire together
into a flow. The flow is persisted as JSON in SQLite / MySQL /
Postgres, executed by a LangChain.js / LlamaIndex.js runtime, and
exposed simultaneously as an embeddable chat widget, an HTTP API
(`POST /api/v1/prediction/<flow-id>`), an MCP server, and a public
share link. The headline use case is the same as Langflow's:
non-engineers (PMs, prompt engineers, domain experts) iterate
on agent / RAG pipelines visually while engineers consume the
JSON flow or the HTTP endpoint from a real codebase.

## 1. Install footprint

- `npm install -g flowise` (Node.js ≥ 18.15). Single binary on
  PATH: `flowise`. `flowise start` boots the React UI + Express
  backend at `http://localhost:3000`. Default DB is SQLite at
  `~/.flowise/database.sqlite`; switch via `DATABASE_TYPE=mysql`
  / `postgres` env vars.
- Docker path: `docker run -d -p 3000:3000 flowiseai/flowise` —
  the official image bundles Node + a writable `/root/.flowise`
  volume; `docker compose up` recipe in `docker/` covers
  Postgres + Flowise + worker queue.
- Optional dependencies are *opt-in per node*: when you drag in
  a Pinecone node, the runtime installs `@pinecone-database/pinecone`
  on first execution and caches it. No Python toolchain required
  unless you use the `Custom Tool` Python sandbox.
- Node count: ~200 nodes covering chat models, completion models,
  embeddings, vector stores, document loaders, retrievers, output
  parsers, memory, agents, tools, utilities, and recently MCP
  client / server nodes.

## 2. Repo + version + license

- Repo: <https://github.com/FlowiseAI/Flowise>
- Latest release: **flowise@3.1.2** (2026-04-14)
- License: **Apache-2.0** with a commercial-use carve-out for
  files under `packages/server/src/enterprise/` —
  <https://github.com/FlowiseAI/Flowise/blob/main/LICENSE.md>
- Default branch: `main`
- Language: TypeScript (Node.js back end, React + React Flow
  front end)

## 3. What it can build

- **Chat-only** apps: prompt template + chat model + memory + an
  embedded chat bubble for a marketing site.
- **RAG**: document loader (PDF / DOCX / Notion / Confluence /
  GitHub / S3 / Drive / web) → text splitter → embedder → vector
  store (Pinecone / Chroma / Qdrant / Weaviate / Milvus / pgvector
  / Mongo Atlas / Redis / Astra / Upstash) → retriever → answer
  chain with citations.
- **Agents**: ReAct, OpenAI Functions, Tool-Calling, Plan &
  Execute, Conversational Retrieval, Hierarchical Multi-Agent
  Supervisor, AutoGPT-style — backed by LangChain.js agents.
- **Tools**: built-in (calculator, web search via Serper / Brave /
  Tavily / Exa, requests, SQL, Python sandbox, browser via
  Playwright, code interpreter), custom JS function nodes, OpenAPI-
  spec-imported tools, and MCP-server tools.
- **Outputs**: chat widget (`<flowise-chatbot>` web component
  embed), HTTP prediction API, streaming SSE, MCP server publishing
  the flow as a callable tool to MCP-aware agent CLIs
  ([`opencode`](../opencode/) / [`claude-code`](../claude-code/) /
  [`crush`](../crush/) / [`fast-agent`](../fast-agent/)).

## 4. When to use it

- You want a **visual canvas as the source of truth for an LLM
  app** so non-engineers can iterate, and the team is **Node-
  first** (TypeScript, Next.js, Express). Pick this over
  [`langflow`](../langflow/) when the host stack is JS, not
  Python.
- You need a **chat widget you can paste into any website** as
  the deliverable — Flowise's `<flowise-chatbot>` embed is the
  shortest path from "PM tweaks the flow" to "live on the
  marketing page".
- You want **internal RAG bots** for a knowledge base with a
  drag-drop editor for ops / support staff to tune the prompt
  and the retriever without filing a PR.
- You want to **prototype an agent** before hand-coding it —
  build it visually, then export the flow JSON or read the
  HTTP API spec and rewrite in LangChain.js with confidence
  the topology works.
- You need **MCP server publishing** so a flow becomes a callable
  tool from any MCP-aware coding agent.

## 5. When NOT to use it

- The team is **Python-first** — pick [`langflow`](../langflow/)
  instead. Same shape, Python runtime (LangChain Python /
  LangGraph), bigger node catalogue for Python-only integrations.
- You want **declarative, optimisable prompt programs with
  auto-prompt-tuning** — that is [`dspy`](../dspy/), not a
  drag-drop graph.
- You want **a code-first agent framework** with end-to-end
  TypeScript types (Zod-typed tool inputs, typed workflow DAG) —
  reach for [`mastra`](../mastra/) instead. Flowise's typing is
  per-node config, not end-to-end TS.
- You want **MCP-purist code-first agent crews** — pick
  [`fast-agent`](../fast-agent/).
- You need **strict licence cleanliness for a redistributed
  commercial product** — note the `enterprise/` directory carve-
  out; consult counsel or strip those files at build time.
- You only need **one chat endpoint with a couple of tools** —
  hand-rolling LangChain.js + Express is shorter than running
  the whole Flowise platform.

## 6. Closest alternatives

- [`langflow`](../langflow/) — Python sibling, near-identical
  visual stance; pick by the host language of your team.
- [`mastra`](../mastra/) — TypeScript-native code-first agent
  framework with typed workflows and a dev-server playground;
  pick when types + code beat drag-drop.
- [`fast-agent`](../fast-agent/) — code-first MCP-native agent
  crews; pick when the value is MCP rigour, not a canvas.
- [`chainlit`](../chainlit/) — code-first chat UI for Python
  agents; pick when the deliverable is the chat surface, not
  the flow editor.
- [`langgraph`](../langgraph/) / [`langroid`](../langroid/) /
  [`crewai`](../crewai/) / [`agno`](../agno/) — Python agent
  frameworks Flowise's runtime and visual flows compete with on
  the agent side.

## 7. Hello-world

```bash
# Local single-binary
npx flowise start
# → http://localhost:3000  (default user/pass set via env)

# Or Docker
docker run -d -p 3000:3000 -v ~/.flowise:/root/.flowise \
  flowiseai/flowise

# After building a flow in the UI, call it via HTTP:
curl -X POST http://localhost:3000/api/v1/prediction/<flow-id> \
  -H "Content-Type: application/json" \
  -d '{"question":"What is in this knowledge base?"}'

# Embed the chat widget on any page:
# <script type="module">
#   import Chatbot from "https://cdn.jsdelivr.net/npm/flowise-embed/dist/web.js"
#   Chatbot.init({ chatflowid: "<flow-id>", apiHost: "http://localhost:3000" })
# </script>
```

## 8. Killer feature, weakness, when to choose

- **Killer feature**: visual flow editor → embeddable chat
  widget + HTTP API + MCP server, in one Node binary, with
  ~200 prebuilt nodes covering the LangChain.js ecosystem and
  ops staff able to tune prompts / retrievers without a PR.
- **Weakness**: drift risk on node configs across releases (pin
  the version), `enterprise/` directory licence carve-out, and
  the visual layer becomes a *source-control liability* once
  the flow is non-trivial — diffing flow JSON in PRs is painful.
- **When to choose**: Node-first team, non-engineer authors of
  LLM flows, embeddable chat widget as the headline deliverable,
  and MCP server publishing as a bonus.

## 9. Repo health (snapshot)

- Very active project (~52k stars), regular weekly releases on
  `main`. Latest **flowise@3.1.2** on 2026-04-14. Monorepo
  managed by `pnpm` + `turbo` with packages for `server`,
  `ui`, `components`, and `embed`.
- Funded company behind the project (FlowiseAI, Inc.) — note
  the commercial carve-out for `packages/server/src/enterprise/`
  in `LICENSE.md`. The rest is Apache-2.0.
- Backwards-compatibility on flow JSON has held across 2.x → 3.x;
  individual node names / config shapes do churn on minor bumps,
  so pin the Flowise version in production deployments.
