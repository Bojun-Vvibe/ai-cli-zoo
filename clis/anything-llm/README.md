# anything-llm

> Snapshot date: 2026-04. Upstream: <https://github.com/Mintplex-Labs/anything-llm>
> Pinned release: `v1.12.1` (HEAD `2029f90b4249e0977695808dfc04e91117accc36`,
> default branch `master`).
> License file: [`LICENSE`](https://github.com/Mintplex-Labs/anything-llm/blob/master/LICENSE)
> (sha `cc42d1d080250fb38c47b81ed8f9fb1d64dc2965`, MIT).

`anything-llm` is Mintplex Labs' all-in-one self-hostable
**document-chat / agent / RAG platform**: drop a folder of
PDFs, web URLs, GitHub repos, Confluence spaces, or Notion
pages into a "workspace", pick an LLM provider and a vector
store, and chat against the corpus from a polished React UI,
a desktop app (Electron), a Chrome extension, an embeddable
web widget, or a JSON HTTP API. It is the catalog's reference
for **a batteries-included self-hosted RAG / agent platform
with a real product surface**, sibling to
[`docsgpt`](../docsgpt/), [`khoj`](../khoj/),
[`open-webui`](../open-webui/), [`librechat`](../librechat/),
[`r2r`](../r2r/), [`ragflow`](../ragflow/),
[`private-gpt`](../private-gpt/),
[`flowise`](../flowise/), [`langflow`](../langflow/).

## 1. Install footprint

- One-click Docker: `docker pull mintplexlabs/anythingllm`
  then `docker run -p 3001:3001 …` boots the full stack
  (Node back-end, React front-end, SQLite db, vector store
  defaults to LanceDB embedded).
- Bare-metal: `yarn dev:server`, `yarn dev:frontend`,
  `yarn dev:collector` across the three workspaces in the
  monorepo (Node 18+, requires Yarn).
- Desktop: a packaged Electron app for macOS / Windows /
  Linux (single-user, embedded vector store, no Docker).
- LLM providers: OpenAI, Anthropic, Gemini, AWS Bedrock,
  Cohere, Mistral, Groq, Together, OpenRouter, Perplexity,
  Hugging Face, Ollama, LocalAI, LM Studio, KoboldCPP,
  text-generation-webui, any OpenAI-compatible endpoint —
  selected per-workspace through the UI.
- Vector stores: LanceDB (default, embedded), Chroma,
  Pinecone, Qdrant, Weaviate, Milvus, AstraDB, Zilliz,
  pgvector — also per-workspace.
- Embedders: OpenAI, native (built-in transformers.js),
  Ollama, LocalAI, LM Studio, Cohere, Voyage, any
  OpenAI-compatible.

## 2. Repo, version, license

- Repo: <https://github.com/Mintplex-Labs/anything-llm>
- Latest release: `v1.12.1` (2026-04-22).
- HEAD pinned at this snapshot:
  `2029f90b4249e0977695808dfc04e91117accc36`.
- License: MIT. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/Mintplex-Labs/anything-llm/blob/master/LICENSE)
  (sha `cc42d1d080250fb38c47b81ed8f9fb1d64dc2965`).
- Hosted flavour: <https://useanything.com> (opt-in cloud);
  this entry pins the self-hostable OSS path.

## 3. What it actually does

The mental model is **workspaces over a corpus**:

```
Workspace
  ├─ Documents (PDF, DOCX, HTML, MD, code, audio→whisper, …)
  ├─ Web URLs / sitemap crawl
  ├─ Connectors (GitHub, GitLab, Confluence, YouTube,
  │              Drupal, Obsidian, …)
  ├─ Vector store namespace
  ├─ LLM / embedder / system prompt overrides
  ├─ Agent skills (built-in + community + custom JS)
  └─ Threads (chat history, branching)
```

The collector worker ingests each source, chunks with a
configurable splitter, embeds with the workspace's embedder,
and writes to the workspace's vector namespace. The chat
back-end runs a hybrid retrieval (vector + keyword) per turn,
re-ranks, and renders citations back into the UI. **Agents**
(`@agent` in chat) layer on tool-use: web search, web
scraping, SQL, save-to-memory, save-file, custom JS skills,
and any MCP server the operator mounts.

API and embed surfaces:

- `POST /api/v1/workspace/{slug}/chat` — JSON chat against a
  workspace.
- Chat-widget JS snippet for embedding into any website.
- Chrome extension that turns the active page into a workspace
  source.
- Desktop deep-link.

## 4. MCP support

Yes (client). The agent runtime mounts external MCP servers as
tool sources alongside the built-in skill catalog —
filesystem, GitHub, Slack, browser, any MCP server in the
operator's `mcp.json` becomes a tool the workspace agent can
call. No first-party MCP server flavour exposes a workspace
outward — for that, point a host agent at the
`/api/v1/workspace/{slug}/chat` endpoint or wrap it in
[`fastmcp`](../fastmcp/).

## 5. Sub-agent model

Light. The platform is workspace-scoped chat + a single
in-loop tool-using agent per turn, not a multi-agent
topology. The agent picks tools, calls them, and feeds
results back into the same conversation; there is no
planner / researcher / coder split shipped out of the box.
For nested-agent shapes, the right move is to treat
`anything-llm` as the *RAG and tool-use surface* and run the
multi-agent loop one level up
([`langgraph`](../langgraph/), [`crewai`](../crewai/),
[`deer-flow`](../deer-flow/), [`autogen`](../ag2/)).

## 6. Telemetry stance

Anonymous usage telemetry is on by default in self-host (the
back-end posts feature-event pings to PostHog with no PII or
chat content); opt-out via `DISABLE_TELEMETRY=true` in the
server env. LLM and embedder providers see their own
traffic per their posture; the desktop app and Docker image
inherit the same toggle. Pair with Ollama + LanceDB +
`DISABLE_TELEMETRY=true` for an air-gapped configuration.

## 7. Token / context strategy

Per-workspace knobs: chunk size, chunk overlap, top-K, similarity
threshold, max tokens per response, "Pin documents in context"
(skip retrieval and stuff a small doc as a system prefix), and
`@agent` mode (turn-by-turn tool-use). Long chats are
summarised with a sliding window the operator can size; the
hybrid retrieval (vector + BM25) reduces the "irrelevant
chunk crowds out the right one" failure mode that plain
top-K vector RAG hits on long-tail queries.

## 8. Hot keybinds

The chat UI is mouse-driven; the desktop app inherits standard
Electron shortcuts. Headless / programmatic users go through
the JSON API or the embed widget.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A self-hostable RAG / agent platform
with a real product surface — multi-source ingestion,
multi-provider model selection, multi-vector-store backend,
agent + MCP tool-use, chat-widget / Chrome / desktop / API
distribution — all in one MIT repo.** Most of the catalog's
RAG layer is "you get an HTTP endpoint, build your own UI";
`anything-llm` ships the UI, the connector catalog, the
permissioned multi-user / multi-workspace model, and the
embed surfaces, so "give the support team a chatbot over
our internal Confluence by next Tuesday" becomes one Docker
run plus a connector wizard.

**Weakness.** It is **a product, not a primitive** — extending
beyond what the workspace + skills model supports (custom
multi-agent topologies, custom retrieval pipelines, custom
ingestion flows) means living inside the Node monorepo
instead of composing against a slim library. Performance at
very large corpora depends heavily on the vector-store
choice; the embedded LanceDB default is fine to a few million
chunks but past that the team ends up running Qdrant /
Milvus / Weaviate alongside, at which point a thinner stack
([`r2r`](../r2r/), [`ragflow`](../ragflow/),
[`docsgpt`](../docsgpt/)) may be a better composition root.

**Choose `anything-llm` when** the goal is "self-hosted
RAG / agent platform with a polished UI and multi-tenant
workspaces, today" — internal knowledge bases, customer
support over a doc corpus, a private ChatGPT replacement
across a small team. **Choose something else when** the
shape is single-user local notes / PDFs daemon
([`khoj`](../khoj/)), the team wants a no-code agent
*builder* surface ([`flowise`](../flowise/),
[`langflow`](../langflow/), [`docsgpt`](../docsgpt/)),
the workload is a multi-agent harness rather than a
chat-over-corpus product ([`langgraph`](../langgraph/),
[`crewai`](../crewai/), [`deer-flow`](../deer-flow/)),
or the priority is a slim composable RAG library
([`r2r`](../r2r/), [`fast-graphrag`](../fast-graphrag/),
[`embedchain`](../embedchain/)).
