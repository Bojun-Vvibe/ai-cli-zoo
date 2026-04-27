# docsgpt

> Snapshot date: 2026-04. Upstream: <https://github.com/arc53/DocsGPT>
> Pinned release: `0.17.0` (HEAD `af618de13d94afb276c34d4d2414574f06a399fd`).
> License file: [`LICENSE`](https://github.com/arc53/DocsGPT/blob/main/LICENSE)
> (sha `eb01d1a0cee26762aad4edfcd97a9d1b9b361857`, MIT).

`docsgpt` is a self-hostable RAG / agent platform that started
as "ChatGPT for your docs" and has grown into a private-cloud
enterprise-search shape: ingest a corpus (web crawl, GitHub
repo, Confluence, Notion, S3, raw uploads), chunk and embed it
into a vector store (FAISS by default; Qdrant / Milvus /
Mongo / Elastic / Pinecone supported), and serve chat / search
through a Flask back-end, a React web app, a Chrome
extension, a Discord bot, and a Chatwoot integration. The
catalog scope is the back-end + bundled `docsgpt` CLI / worker
processes that drive ingestion and serving — the surface a
developer who wants to "stand up a private RAG service"
actually touches. It is the catalog's reference for **a
batteries-included self-hostable RAG agent platform with a
built-in agent builder and document analysis loop**, sitting
in the same niche as [`r2r`](../r2r/), [`ragflow`](../ragflow/),
[`khoj`](../khoj/), [`private-gpt`](../private-gpt/),
[`paper-qa`](../paper-qa/), and [`dify`](../dify/) — but
biased toward enterprise-search ergonomics (multi-tenant
sources, agent builder UI, Chrome / Chatwoot front-ends) over
chat-only or research-paper workflows.

## 1. Install footprint

- `git clone arc53/DocsGPT && ./setup.sh` brings up the full
  Docker Compose stack (back-end, front-end, MongoDB, Redis,
  Celery worker) on `:5173` (web) / `:7091` (API).
- Cloud-managed: <https://app.docsgpt.cloud/>.
- Python: the back-end is a Flask app under `application/`
  with a Celery worker for async ingestion; `pip install -r
  application/requirements.txt` then `flask run` is the
  source-route alternative to Docker.
- Front-ends: `frontend/` (React web app), `extensions/`
  (Chrome / VS Code / Discord / Chatwoot bridges) — each
  builds independently and points at the back-end URL.
- LLM defaults: OpenAI for chat + embeddings; switch to local
  via `LLM_PROVIDER=docsgpt` (bundled tiny model),
  `LLM_PROVIDER=ollama`, `LLM_PROVIDER=llama.cpp`, or any
  OpenAI-compatible endpoint via `OPENAI_BASE_URL`.

## 2. Repo, version, license

- Repo: <https://github.com/arc53/DocsGPT>
- Latest release: `0.17.0` (2026-04-21).
- HEAD pinned at this snapshot:
  `af618de13d94afb276c34d4d2414574f06a399fd`.
- License: MIT. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/arc53/DocsGPT/blob/main/LICENSE)
  (sha `eb01d1a0cee26762aad4edfcd97a9d1b9b361857`).

## 3. What it actually does

The platform is shaped as an **agent + sources + tools**
triple, configurable from the web UI ("Agent Builder") or the
HTTP API:

- **Sources.** Upload PDFs / DOCX / Markdown / HTML / images
  (OCR via Tesseract), point at a GitHub repo, schedule a web
  crawl, or connect Confluence / Notion / Google Drive / S3 /
  Azure Blob / Mongo / Postgres. Ingestion runs in Celery —
  chunked, embedded, written to the configured vector store.
- **Agents.** A named bundle of (system prompt, source
  selection, tool selection, model). Agent Builder exposes a
  no-code form; the same agent is callable over `/api/answer`.
- **Tools.** Built-in catalog: web search (Brave / SerpAPI /
  DuckDuckGo / Tavily), Python execution (sandboxed), API
  callout (declared by OpenAPI spec), MCP tool mounts,
  Read-Aloud (TTS), image generation (DALL-E / SD endpoints),
  scheduled "deep research" loops.
- **Deep research.** Multi-step plan-search-read-cite agent
  loop that iterates source queries until a configurable
  budget (depth or token limit) is exhausted; emits a cited
  Markdown report similar in shape to
  [`local-deep-research`](../local-deep-research/) but wired
  into your private corpus.

## 4. MCP support

Yes (client). The agent builder lets you mount external MCP
servers as tools — the agent's tool registry includes any
declared MCP endpoints alongside the built-in tools, with the
same JSON-schema arg shape. No first-party MCP server flavour
that exposes the DocsGPT corpus to an outside agent (a small
Flask shim would be the diff).

## 5. Sub-agent model

Implicit in the deep-research tool — the planner LLM emits
sub-questions, each sub-question is a separate
search-and-read sub-call against the source set, and a
synthesis pass collapses them into the final answer. Not
exposed as user-facing nested-agent CLI; the abstraction is
"the agent has tools, one of which happens to be a small
internal research loop".

## 6. Telemetry stance

Off by default in OSS self-host. The Flask back-end does not
phone home; LLM and embedding providers see prompts +
documents per their own posture (set `LLM_PROVIDER=ollama`
plus a self-hosted vector store for fully-air-gapped). The
hosted `app.docsgpt.cloud` tier is the opt-in cloud surface.
MongoDB stores conversations, sources, and agent definitions
locally.

## 7. Token / context strategy

Per-agent. Chunk size, top-k retrieval, reranker on/off,
max-context-tokens, and history truncation are agent-level
fields. Long-document analysis uses **map-reduce summarisation**
behind the scenes — the document is chunked, each chunk
summarised, summaries combined — so very long PDFs do not
require a long-context model. The deep-research loop has its
own per-iteration budget and a hard total cap.

## 8. Hot keybinds

None at the back-end / CLI; the React web app has the
expected chat-shaped shortcuts (`Enter` send, `Shift-Enter`
newline, `Esc` cancel current generation). The CLI surface is
plain `flask` / `celery` / `docker compose` ops.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A no-code agent builder layered on top
of a real source-and-vector-store ingestion pipeline, with a
ready-made Chrome extension and Chatwoot bridge for
non-technical end-users.** Most catalog RAG tools stop at "you
get an HTTP endpoint, build your own UI"; DocsGPT ships the
UI, the multi-source ingestion, the agent builder, and the
distribution surfaces (web app + Chrome + Discord + Chatwoot)
in one MIT-licensed repo. For a team that needs "let support
agents query our docs corpus from inside their existing
Chatwoot helpdesk" the integration budget is closer to one
afternoon than one quarter, and the same back-end serves the
internal "ask the docs" Slack-shaped use case from the same
agent definition.

**Weakness.** The breadth comes at a cost: the codebase is a
mid-size Flask + React + Celery + Mongo + Redis stack (not a
small library you embed), and the Docker Compose footprint is
hefty for a hobby deploy. Several integrations
(Chatwoot, Discord, some enterprise sources) are minimally
documented and assume you already know the target platform.
Eval / observability is thin compared to dedicated tools —
plug [`langfuse`](../langfuse/) or [`helicone`](../helicone/)
in front of the model calls if you need request-level tracing.

**Choose `docsgpt` when** the goal is "stand up a private
multi-source RAG service with a real UI, an agent builder, and
end-user surfaces (web / Chrome / Discord / Chatwoot)" and you
want one MIT repo to own the whole vertical. **Choose
something else when** the requirement is a research agent over
academic papers ([`paper-qa`](../paper-qa/)), a heavyweight
RAG framework you embed in your own product
([`r2r`](../r2r/), [`ragflow`](../ragflow/)), a
visual-flow-builder LLM platform ([`dify`](../dify/),
[`flowise`](../flowise/)), a personal local assistant
([`khoj`](../khoj/), [`private-gpt`](../private-gpt/)), or a
fully local research loop with no SaaS surface
([`local-deep-research`](../local-deep-research/)).
