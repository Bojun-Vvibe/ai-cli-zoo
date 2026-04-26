# dify

> Snapshot date: 2026-04. Upstream: <https://github.com/langgenius/dify>

A self-hostable, web-first **agentic-workflow platform**. You compose
prompts, retrieval pipelines, tool calls, and conditional branches in a
visual canvas, then expose the resulting agent over an HTTP / SDK API
your own product can call. The canvas surface is the headline feature
and the reason to pick it over a code-first framework: a non-engineer
can wire RAG → router → tool-call → answer in an afternoon, and the
runtime that executes it is the same one your service will hit in
production.

In the catalog it sits next to [`langflow`](../langflow/) and
[`flowise`](../flowise/) — all three are visual workflow builders —
but `dify` carries the most complete *operations* surface (per-tenant
quotas, dataset versioning, eval, log viewer, app analytics) at the
cost of a heavier deploy footprint (Docker Compose with Postgres +
Redis + Weaviate/Qdrant + a sandbox container).

## 1. Repo facts

- Repository: <https://github.com/langgenius/dify>
- Latest release: **v1.13.3** (2026-03-27)
- Primary language: TypeScript (frontend) + Python (backend)
- License: **Modified Apache-2.0** with multi-tenant + branding
  restrictions — see
  [`LICENSE`](https://github.com/langgenius/dify/blob/main/LICENSE)
  (blob SHA `329ee302875f292fdd11e12e830562d0be9ade1c`). You may
  self-host and embed; you may **not** run it as a multi-tenant SaaS
  without a separate commercial license, and you may not strip the
  console branding.

## 2. Why it matters

- **Workflow canvas with a real runtime behind it.** Unlike most
  no-code agent builders, what you draw in the editor is what the
  HTTP API executes — no codegen step, no drift between design and
  deploy.
- **Dataset + RAG are first-class.** Upload a folder, pick a chunker
  and an embedder, point an app at the dataset; reindex on schedule
  without touching code.
- **Per-app observability built in.** Every conversation is logged
  with token cost, latency, retrieval hits, and tool-call traces, and
  the same data backs the eval / annotation surface.

## 3. Install

The supported install is Docker Compose. There is no single-binary
CLI; you operate `dify` by bringing the stack up and using the web
console or the API.

```sh
git clone https://github.com/langgenius/dify.git
cd dify/docker
cp .env.example .env
docker compose up -d
# console at http://localhost/install for first-run setup
```

## 4. Example invocation

Once an app is published in the console you call it as a normal HTTP
endpoint with the per-app API key:

```sh
curl -X POST 'http://localhost/v1/chat-messages' \
  -H "Authorization: Bearer $DIFY_APP_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "inputs": {},
    "query": "summarize the latest release notes",
    "response_mode": "streaming",
    "user": "abc-123"
  }'
```

## 5. When to pick something else

- You want a **code-first** agent framework with a Python API →
  [`langgraph`](../langgraph/), [`crewai`](../crewai/),
  [`agno`](../agno/).
- You want a **lighter** visual builder without the ops surface →
  [`langflow`](../langflow/) or [`flowise`](../flowise/).
- You want a chat UI you can ship to end users without the workflow
  canvas → [`librechat`](../librechat/) or
  [`open-webui`](../open-webui/).
