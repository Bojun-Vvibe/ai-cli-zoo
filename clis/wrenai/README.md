# wrenai

> Snapshot date: 2026-04. Upstream: <https://github.com/Canner/WrenAI>
> License file: <https://github.com/Canner/WrenAI/blob/main/LICENSE>
> (AGPL-3.0).
> Pinned: `0.29.1` (latest tagged release).

An **open-source text-to-SQL + text-to-chart GenBI agent** built
around a **semantic layer** (Wren's MDL — Modeling Definition
Language). You describe your warehouse once as typed entities +
relationships + measures, and Wren turns "what was Q1 NRR for the
APAC enterprise segment" into accurate SQL, runs it against the
warehouse, and renders a chart — without leaking raw schema to the
LLM. Sits in the same neighborhood as [`vanna`](../vanna/) (RAG over
DDL), [`dataline`](../dataline/), and [`defog`](../defog/)-class
products, with the **semantic layer + multi-source dialect support
+ MDL versioning** as the differentiator.

## 1. Install footprint

- **Launcher (recommended)**: `curl -L
  https://github.com/Canner/WrenAI/releases/latest/download/wren-launcher-darwin.tar.gz
  | tar xz && ./wren-launcher-darwin` — interactive setup, picks an
  LLM provider, and `docker compose`-ups the full stack: `wren-ui`
  (Next.js), `wren-ai-service` (Python / FastAPI / Haystack
  pipelines), `wren-engine` (Rust query rewriter + connector layer),
  `qdrant` (vector store), Postgres (app metadata), and Ibis
  (Python-side dialect translation for the connector matrix).
- **Manual**: `git clone https://github.com/Canner/WrenAI && cd
  WrenAI/docker && docker compose --env-file .env up`. Expect ~3 GB
  of images, ~1.5 GB RSS at idle. Open `http://localhost:3000` for
  the modeling + chat UI.
- **CLI surface**: the `wren-launcher` binary is the entry point
  (start / stop / upgrade / health-check); day-to-day driver is
  the web UI plus the REST / GraphQL API on port `:5556`. No
  per-query `wren ask "..."` shell mode.
- **Dependencies**: Docker + Docker Compose v2. LLM provider key
  (OpenAI / Anthropic / Gemini / Ollama / any LiteLLM-supported).

## 2. Repo + version + license

- Repo: <https://github.com/Canner/WrenAI>
- Latest release: **`0.29.1`**
- License: **AGPL-3.0** —
  <https://github.com/Canner/WrenAI/blob/main/LICENSE>. The AGPL
  copyleft applies if you offer Wren as a network service to third
  parties; internal-use deployments are unaffected. Canner offers a
  separate commercial license for embedding.
- Default branch: `main`
- Languages: TypeScript (UI), Python (AI service), Rust (engine), Go
  (launcher)

## 3. Models supported

Routed through `litellm` plus first-class adapters: OpenAI (GPT-4o
/ 4.1 / o-series), Anthropic (Claude 3.5 / 3.7 / 4 Sonnet),
Google (Gemini 1.5 / 2.x via AI Studio + Vertex), AWS Bedrock,
Mistral, Cohere, Groq, DeepSeek, Together, Fireworks, OpenRouter,
Ollama (local — useful for the SQL-generation hop where prompts are
schema-heavy and tokens add up), llama.cpp, vLLM, any
OpenAI-compatible endpoint. Embeddings used for the
schema-retrieval hop: OpenAI, Cohere, VoyageAI, sentence-transformers
locally. Vector store is Qdrant by default.

**Data sources** (Wren's other matrix): PostgreSQL, MySQL, SQLite,
DuckDB, BigQuery, Snowflake, Redshift, Trino / Presto, Athena,
Databricks (Unity Catalog), ClickHouse, MS SQL, plus CSV / Parquet
files. Dialect translation is owned by `wren-engine` (Rust + Ibis),
so the LLM sees one **canonical SQL dialect** keyed off the MDL,
and the engine rewrites to the target warehouse's dialect — the
LLM never has to know the difference between `STRING_AGG` and
`LISTAGG`.

## 4. MCP support

Yes (server). Wren ships a first-party MCP server
(`wren-ai-service` exposes an MCP transport) so external agents
([`claude-code`](../claude-code/), [`opencode`](../opencode/),
[`cline`](../cline/), [`continue`](../continue/)) can ask
"give me Q1 revenue by region" and get back rows + chart spec
without re-implementing the semantic layer. This is the most
useful integration shape for Wren — let the existing coding agent
own the conversation, let Wren own the warehouse.

## 5. Sub-agent model

Pipeline-shaped, not crew-shaped. The Haystack-based AI service
runs a fixed graph: **(intent classifier) → (semantic-layer
retriever — entities, relationships, measures relevant to the
question) → (SQL generator — produces canonical-dialect SQL against
the MDL) → (SQL corrector — tries the engine, repairs on parse /
plan errors) → (chart-spec generator — Vega-Lite over the result
set) → (summary writer)**. Each stage is a swappable component;
there is no "spawn a planner agent that decides what to do" loop,
which is intentional — the value proposition is **predictable,
auditable text-to-SQL with a semantic layer in the middle**, not
an open-ended agent.

## 6. Telemetry stance

Off by default. Fully self-hosted; egress is the configured LLM /
embedding provider and the user's own warehouse. Optional
Langfuse-backed tracing of the pipeline is opt-in — point
`LANGFUSE_*` env vars at your own [`langfuse`](../langfuse/)
instance and every retrieval / SQL-gen / SQL-correction hop is
captured as a span.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **The MDL semantic layer is the LLM's input,
not your raw schema** — and that single design choice is what makes
Wren accurate where naive text-to-SQL is brittle. You define
entities (`Customer`, `Order`), relationships (`Customer 1..N
Order`), and measures (`monthly_recurring_revenue := SUM(...)`)
once, version them in git, and the LLM's prompt becomes "here are
the typed concepts you can use; write SQL using only those." The
Rust engine then rewrites the canonical SQL to whichever of
12+ warehouse dialects you actually run on, validates against the
plan, and returns rows + a Vega-Lite chart spec. The MCP server
makes the same capability available to any catalog-CLI agent: a
`claude-code` session can call `wren.ask("revenue by region last
quarter")` mid-conversation and stream back a typed answer the
agent can reason over. Multi-source joins through the MDL layer
work because the engine, not the LLM, owns dialect translation.

**Weakness.** **AGPL + heavy stack.** Six containers, ~3 GB of
images, Postgres + Qdrant + an LLM endpoint to keep alive — Wren
is not a single static binary the way [`shell-gpt`](../shell-gpt/) or
[`tlm`](../tlm/) is. AGPL-3.0 means SaaS-style redistribution
requires either source disclosure or a commercial license from
Canner; teams building closed-source BI products will need to
read the license carefully or pay. The MDL has a learning curve —
"text-to-SQL just works" is true after you model your warehouse, not
before, and that modeling step is real (hours to days for a
non-trivial schema). Latency: each question is a multi-hop pipeline
(intent → retrieve → generate → correct → chart), so end-to-end is
seconds, not subsecond.

**When to choose.** You are building a **GenBI / "ask your
warehouse" surface** for non-engineers and you have, or are
willing to build, a real semantic layer. You want **multi-warehouse
support without re-prompting per dialect**, **chart output as a
first-class artifact, not a screenshot**, and an **MCP endpoint
your existing coding agent can call**. Pair with
[`vanna`](../vanna/) if you want a lighter Python-only RAG-over-DDL
fallback for warehouses you are not ready to model, with
[`langfuse`](../langfuse/) for production tracing of the pipeline,
and with any catalog coding CLI that speaks MCP for the
"agent-asks-the-warehouse-mid-task" workflow. Skip if your need is
"single-database NL-to-SQL with no semantic layer" — Vanna or a
direct LLM call against the schema is lighter — or if AGPL is a
non-starter for your distribution model.
