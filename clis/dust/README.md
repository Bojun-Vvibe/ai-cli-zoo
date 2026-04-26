# dust

> Snapshot date: 2026-04. Upstream: <https://github.com/dust-tt/dust>

**Custom AI agent platform for building and deploying assistants over your
team's data sources.** Dust is a TypeScript monorepo that ships a workspace-
shaped product (Connections + Data Sources + Assistants + Conversations) with
a CLI / SDK surface for building "GPTs over your Slack / Notion / Google
Drive / GitHub / Confluence / Intercom". The platform piece is the moat:
managed connectors with incremental sync, per-data-source ACL inheritance,
hybrid (BM25 + dense) retrieval, and a typed `Dust Apps` DSL that chains
LLM calls, retrieval, and code execution into reusable assistants you can
publish across the workspace.

## Repo + version + license

- Repo: <https://github.com/dust-tt/dust>
- Latest release: **`dsbx-v0.1.4`** (2026-04-16)
- HEAD on `main`: `98533b9`
- License: **MIT** —
  <https://github.com/dust-tt/dust/blob/main/LICENSE>
- License path in repo: `LICENSE` (1058 bytes, standard MIT text)
- Default branch: `main`
- Language: TypeScript (Node + Next.js + Rust core for the dust-apps engine)

## Install

```bash
# Self-host via the front-end + core monorepo
git clone https://github.com/dust-tt/dust && cd dust
# follow front/README.md and core/README.md for Postgres + Qdrant + Temporal setup
# OR use the hosted product at https://dust.tt and the SDK only:
npm install @dust-tt/client
```

```ts
import { DustAPI } from "@dust-tt/client";
const api = new DustAPI(
  { url: "https://dust.tt" },
  { workspaceId: "...", apiKey: process.env.DUST_API_KEY! },
  console,
);
const r = await api.createConversation({
  title: null, visibility: "unlisted",
  message: { content: "Summarise the last week of #engineering", mentions: [{ configurationId: "asst_..." }] },
});
```

## Niche

The "**workspace-shaped agent platform with first-party connectors and
RBAC inheritance**" slot. Where [`flowise`](../flowise/) and
[`langflow`](../langflow/) give you a graph IDE for building one-off agents,
and where [`composio`](../composio/) hands you OAuth-wrapped tool schemas to
plug into any framework, Dust is opinionated about the *product* shape: a
team workspace with admin / builder / user roles, managed connectors that
stay incrementally in sync without you running a worker pool, retrieval
that respects the source system's ACLs (a user querying Drive only ever
sees chunks from files they could open in Drive), and assistants
publishable to channels / Slack / API. The CLI / SDK is for ops and
embedding, not for "build me an agent from scratch in code".

## Why it matters

- **Managed connectors with ACL inheritance** — Slack / Notion / Google
  Drive / GitHub / Confluence / Intercom / Salesforce / Zendesk / Snowflake
  ship with incremental sync (Temporal-orchestrated), per-document ACL
  capture, and per-user query-time filtering. Most "GPT over my docs"
  products either ignore ACLs or re-implement them — Dust mirrors them.
- **Dust Apps as a typed agent DSL** — JSON-shaped blocks (`llm`,
  `data_source`, `code`, `chat`, `curl`, `database`) compose into
  versioned, parameterised, runnable apps with a Rust execution core
  (`core/`); same app runs from the workspace UI, the SDK, and as the
  backing implementation of an Assistant.
- **Multi-provider model routing** — Anthropic Claude, OpenAI, Google
  Gemini, Mistral, Together, Fireworks, plus Azure OpenAI; per-assistant
  model assignment so the planner can be Claude Opus and the worker can
  be Mistral Small without rewriting the assistant.
- **MIT licence on the whole monorepo** — including connectors, the Rust
  core, and the front-end; you can self-host the entire workspace product
  without an EE carve-out (operationally heavy: Postgres + Qdrant +
  Temporal + Redis + ElasticSearch are all required).
