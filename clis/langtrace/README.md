# langtrace

> Snapshot date: 2026-04. Upstream: <https://github.com/Scale3-Labs/langtrace>

"**Open-source, OpenTelemetry-based end-to-end observability for LLM
applications.**" Langtrace is a self-hostable trace collector + web
dashboard plus first-party Python and TypeScript SDKs. The pitch: every
LLM / vector-DB / framework call is captured as a **standards-grade
OTel span** with the **OpenLLMetry semantic conventions**
(`gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`,
`gen_ai.usage.output_tokens`, `db.system`, `db.collection.name`), so
the same spans render in Langtrace's bundled UI *and* land in Jaeger /
Tempo / Honeycomb / Datadog / SigNoz / Grafana via standard OTLP
export — no vendor-shaped span schema, no second instrumentation pass.

## 1. Install footprint

- SDK: `pip install langtrace-python-sdk` (~10 MB) or
  `npm install @langtrase/typescript-sdk`. One line:
  `from langtrace_python_sdk import langtrace; langtrace.init(api_key=...)`
  before any LLM call wires up the auto-instrumentors.
- Self-hosted backend: `git clone && docker compose up` brings up the
  Next.js dashboard + Postgres + ClickHouse trace store + Redis on
  `localhost:3000`; or use the hosted `langtrace.ai` SaaS by passing
  the API key only.
- Python ≥ 3.9 / Node ≥ 18; SDK pulls `opentelemetry-sdk` +
  `opentelemetry-exporter-otlp-proto-http` + colorama.

## 2. Repo + version + license

- Repo: <https://github.com/Scale3-Labs/langtrace>
- Latest release: **v4.0.11** (2026)
- License: **AGPL-3.0** (the dashboard / server) —
  <https://github.com/Scale3-Labs/langtrace/blob/main/LICENSE>
- SDK packages (`langtrace-python-sdk`, `@langtrase/typescript-sdk`)
  are **Apache-2.0** in their own repos — the AGPL covers the
  *server*, not the instrumentation you embed in your app
- Default branch: `main`
- Language: TypeScript (Next.js dashboard) + Python / TS SDKs

## 3. Auto-instrumentations shipped

LLM SDKs: OpenAI, Anthropic, Cohere, Groq, Mistral, Gemini / Vertex,
Bedrock, Ollama, Perplexity, xAI, DeepSeek, Together, AWS / Azure
OpenAI. Frameworks: LangChain, LangGraph, LlamaIndex, DSPy, CrewAI,
AutoGen, Vercel AI SDK, Haystack, LiteLLM, Pydantic-AI. Vector
stores: Pinecone, Qdrant, Chroma, Weaviate, Milvus, pgvector. Each
auto-instrumentor emits OpenLLMetry-conformant spans so the *same*
`gen_ai.*` attribute names appear regardless of which provider /
framework produced the call — uniform query surface across a polyglot
stack.

## 4. Eval + prompt-management surface

- **Annotations + evaluations**: open a span in the UI, score it
  against a configurable rubric (factuality, helpfulness, toxicity,
  custom), the score is back-attached to the span and queryable as a
  filter ("show me last week's traces with helpfulness < 3").
- **Datasets**: select traces from the UI → export as a JSONL dataset
  → re-run a new prompt against the same inputs → side-by-side diff.
- **Prompt registry**: versioned prompts with deploy labels
  (`production` / `staging`); SDK fetches the live version at runtime
  so prompt changes do not require a code deploy.
- **Playground**: ad-hoc prompt + model + parameter experiment surface
  inside the dashboard, results writeable into a dataset.

## 5. Telemetry stance

SDK is opt-in on `langtrace.init(api_key=...)`; without the key the
SDK is a no-op. The hosted `langtrace.ai` backend sees prompts +
completions you ship to it (that is the product). The self-hosted
backend ships everything to your own ClickHouse — no phone-home from
the dashboard or the SDK to Scale3 once self-hosted. Set
`LANGTRACE_BATCH=true` + `disable_logging=True` for sensitive
workloads, and use the SDK's `with_additional_attributes(...)` +
content-redactor hooks to drop sensitive fields before export.

## 6. Killer feature, weakness, when to choose

**Killer feature.** **OTel + OpenLLMetry as the substrate, not as a
side-export** — every span Langtrace produces uses the upstream
OpenLLMetry semantic conventions, so the *same* spans flow into the
bundled Langtrace UI **and** into Jaeger / Tempo / Honeycomb / Datadog
/ SigNoz / Grafana via plain OTLP. You get the LLM-aware UI when you
want it (token + cost rollups, per-prompt-version comparison, dataset
extraction, RAG-trace grouping with retriever / reranker / generator
spans rendered as a tree) *and* the LLM spans land in the trace
backend the rest of your microservices already use, so an SRE can
search "5xx from the checkout API correlated with `gen_ai.system =
openai` slow spans" in one Grafana query — no parallel observability
stack to maintain. The bundled dashboard ships prompt management
(versioning + deploy labels), evaluations (manual + LLM-as-judge),
and dataset curation (sample from production traces to build eval
sets) in one self-hostable AGPL service, instead of stitching three
separate tools.

**Weakness.** **AGPL on the server is a real licensing trap** if your
plan is "fork the dashboard, run it as a hosted product for our
customers, do not contribute back" — that triggers AGPL §13 (network
copyleft) and you owe source. Internal use inside a company is fine
(AGPL §13's "user interacting with it remotely through a network" is
satisfied by giving employees the source on request, which most
companies already do). Apache-2.0 SDKs sidestep this for the
*instrumentation* hop — your application code stays under whatever
licence you want. The product is younger and less battle-tested than
[`langfuse`](../langfuse/) / [`arize-phoenix`](../arize-phoenix/) on
the "10k req/s production fleet" axis; pin the version and load-test
ClickHouse retention before adopting at scale.

**When to choose.** You want **one self-hostable LLM observability
stack with a real product UI *and* OTel-native ingest** so that LLM
spans land in the *same* Tempo / Honeycomb / Datadog backend the rest
of your stack already uses, plus a bundled prompt-management +
evaluation + dataset workflow without standing up
[`langfuse`](../langfuse/) + [`promptfoo`](../promptfoo/) +
[`argilla`](../argilla/) separately. Pick over `langfuse` (MIT) when
the OTel-conformant `gen_ai.*` span shape matters more than MIT
licensing — Langfuse's tracing is great but it's Langfuse-shaped, not
OpenLLMetry-shaped. Pick over [`arize-phoenix`](../arize-phoenix/)
(ELv2) when AGPL is more acceptable to your legal team than ELv2's
"you may not provide as a hosted service" clause (different traps,
pick the one that fits your business). Pick over
[`openllmetry`](../openllmetry/) when you want a bundled UI on top of
the same span schema (OpenLLMetry is the convention library + the
instrumentors; Langtrace ships the convention + the instrumentors +
the dashboard + the eval / prompt / dataset UI all in one). Skip if
your team already runs Langfuse / Phoenix in prod and the marginal
"OTel-conformant span shape" win is not worth the migration; or if
you only need SDK-side tracing without a control plane (use the
upstream OpenLLMetry / OpenInference packages directly).
