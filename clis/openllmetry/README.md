# openllmetry

> Snapshot date: 2026-04. Upstream: <https://github.com/traceloop/openllmetry>
> License file: <https://github.com/traceloop/openllmetry/blob/main/LICENSE>
> Pinned: **v0.60.0** (2026-04-19, PyPI: `traceloop-sdk`).
> Companion docs: <https://traceloop.com/docs/openllmetry>.

OpenLLMetry is **OpenTelemetry for LLM applications, packaged as a
single SDK plus ~30 auto-instrumentation libraries**, each living in
its own subpackage under `packages/`. The thesis is unfashionable
and correct: LLM observability is not a new domain, it is OTel
spans with a `gen_ai.*` semantic-convention namespace. The OSS
project is a contributor to the OpenTelemetry GenAI semantic
conventions work; the wire shape it emits is the same OTLP that
Jaeger, Tempo, Datadog, NewRelic, Honeycomb, Grafana Cloud, and
SigNoz already understand.

The `traceloop` CLI in `packages/sample-app/` is the developer
ergonomics surface: bootstrap a project, point it at an OTLP
collector (or the hosted Traceloop backend), and run any Python
script with `traceloop run <script.py>` to inherit auto-instrument
on every supported provider import.

## 1. Install footprint

- Core: `pip install -U traceloop-sdk` (or
  `uv pip install traceloop-sdk`). Python 3.10+. Pulls
  `opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-otlp`,
  `posthog`, `colorama`, `tenacity`, `jinja2`, `deprecated`. Slim
  install ~50 MB; one extra per provider you use
  (`opentelemetry-instrumentation-openai`,
  `opentelemetry-instrumentation-anthropic`, etc.).
- Bootstrap call is one line:
  `Traceloop.init(app_name="my-service", api_endpoint="http://localhost:4318")`.
- Auto-instrument list (one subpackage each, all at v0.60.0):
  OpenAI, Anthropic, Cohere, Mistral, Bedrock, Replicate, Vertex AI,
  Google Generative AI, Together, Groq, Watsonx, Ollama,
  Transformers, SageMaker, Alephalpha, VoyageAI, OpenAI Agents SDK;
  vector stores (Chroma, Pinecone, Qdrant, Milvus, Weaviate,
  Marqo, LanceDB); orchestrators (LangChain, LlamaIndex, Haystack,
  CrewAI, Agno); plus an `opentelemetry-instrumentation-mcp` for
  Model Context Protocol traffic.
- Backend: any OTLP-compatible collector. Defaults to gRPC `:4317`
  / HTTP `:4318`. Hosted plane (`api.traceloop.com`) is opt-in via
  `TRACELOOP_API_KEY`.
- Node parity: `traceloop/openllmetry-js` (separate repo) ships the
  same instrumentation surface for TS / Node services.

## 2. License

Apache-2.0 (verified: repo root `LICENSE`, 11357 bytes, full
Apache-2.0 text). Every `packages/opentelemetry-instrumentation-*`
subpackage inherits the same root license; `pyproject.toml` for
each declares `license = "Apache-2.0"`.

## 3. Models supported

OpenLLMetry does not call models — it instruments the libraries
that do. Coverage by ecosystem:

- **Chat / completions:** OpenAI, Anthropic, Cohere, Mistral,
  Bedrock, Vertex AI, Google Generative AI, Together, Groq,
  Watsonx, Ollama, Replicate, SageMaker, Alephalpha, VoyageAI,
  Transformers, OpenAI Agents SDK.
- **Vector stores:** Chroma, Pinecone, Qdrant, Milvus, Weaviate,
  Marqo, LanceDB.
- **Orchestrators:** LangChain, LlamaIndex, Haystack, CrewAI, Agno.
- **Protocol-level:** MCP (client + server traffic).

Spans follow the OTel GenAI semantic conventions: `gen_ai.system`,
`gen_ai.request.model`, `gen_ai.response.model`,
`gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`,
`gen_ai.completion`, plus a custom `traceloop.entity.input` /
`traceloop.entity.output` for prompt-template-aware workflows.

## 4. MCP support

**Yes — instrumentation only.**
`opentelemetry-instrumentation-mcp` wraps both the MCP client and
server transports (stdio, HTTP/SSE) and emits a span per tool
invocation, list-tools call, and resource fetch. Useful for
debugging "why did the agent's MCP tool round-trip cost 4 seconds"
without printf-tracing the JSON-RPC layer. Not an MCP server in
itself.

## 5. Sub-agent model

**N/A — observability layer, not an agent.** Composes naturally
with multi-agent frameworks: a [crewai](../crewai/) crew or a
[pydantic-ai](../pydantic-ai/) graph emits one span per agent step
with parent-child links following the actual call hierarchy, so
the trace shows planner → researcher → coder spans nested as you
would expect.

## 6. Telemetry stance

**Two surfaces. Be precise about which one.**

- *Application telemetry* (the spans the SDK emits about the LLM
  calls in your service): **off by default to anywhere except the
  OTLP collector you configure**. No spans leave your network unless
  you set `TRACELOOP_BASE_URL` (hosted) or
  `OTEL_EXPORTER_OTLP_ENDPOINT` (your own collector).
- *SDK self-telemetry* (PostHog events about how *you* are using
  the OpenLLMetry SDK itself): **on by default**, opt-out with
  `TRACELOOP_TELEMETRY=false` or `Traceloop.init(telemetry_enabled=False)`.
  This is the same posture as [mem0](../mem0/) and a recurring
  pattern in this catalog — read the env-var stance before deploying
  to a regulated environment.

The hosted Traceloop backend is opt-in. Self-host with any
OTLP-compatible collector (Jaeger, Tempo, SigNoz, OpenObserve,
Phoenix from Arize, Datadog OTLP intake, Honeycomb, Grafana Cloud,
NewRelic) for full data sovereignty.

## 7. Prompt-cache strategy

None — observability layer. The instrumentation does record cache
hits surfaced by the underlying provider (Anthropic
`cache_creation_input_tokens` / `cache_read_input_tokens`,
OpenAI `prompt_tokens_details.cached_tokens`) as span attributes
so you can answer "what fraction of my Anthropic spend last week
was cache hits" from a Tempo or Jaeger query without writing
custom logging.

## 8. Hot keybinds (CLI)

Flat command surface (no TUI). Useful one-shots:

| Command | Action |
|---------|--------|
| `traceloop init` | Scaffold `.traceloop/` config in the cwd |
| `traceloop run <script.py>` | Run a Python script with auto-instrument enabled |
| `traceloop run --app-name <name> -- python -m mymodule` | Same with explicit service name |
| `traceloop login` | OAuth-style auth against the hosted backend |
| `traceloop projects list` / `create` | Manage hosted projects |
| `traceloop dataset push <file>.jsonl` | Upload a span-derived dataset for prompt eval |
| `traceloop --version` | Print SDK + bundled instrumentation versions |

## 9. Killer feature, weakness, when to choose

- **Killer:** **OTel as the wire format means the data is portable
  on day one.** Spans land in any OTLP collector; the dashboards
  you already have for HTTP latency, queue depth, and error rate
  also work for `gen_ai.usage.input_tokens` and
  `traceloop.entity.input` once you wire one PromQL / Tempo query.
  The instrumentation matrix (~30 packages) covers the long tail
  most "LLM observability" SaaS products lock to their own SDK,
  including vector stores (Chroma / Pinecone / Qdrant / Milvus /
  Weaviate / Marqo / LanceDB) and orchestrators ([crewai](../crewai/),
  LangChain, LlamaIndex, Haystack, Agno) — a multi-component RAG
  pipeline becomes one trace without you instrumenting the seams.
- **Weakness:** **two-package install tax per provider**. You add
  `traceloop-sdk` *and* `opentelemetry-instrumentation-openai` *and*
  `opentelemetry-instrumentation-anthropic` *and*
  `opentelemetry-instrumentation-pinecone`, version-pinned together
  or version-skew bites at import time. The hosted dashboard is
  pleasant but is a separate paid SaaS; running self-hosted means
  also operating an OTel collector + Tempo / Jaeger / SigNoz, which
  is real ops work that a single-vendor LLM observability SaaS hides
  from you.
- **Choose it when:** you already run OTel for the rest of your
  service (HTTP / DB / queue spans go to Tempo or Jaeger or
  Datadog) and want LLM calls to land in the *same* trace next to
  the HTTP request that triggered them. Or when "no vendor lock-in"
  is a hard procurement constraint and a proprietary LLM
  observability SDK is off the table. Skip it if your stack is
  greenfield, you have no existing OTel pipeline, and an opinionated
  hosted SaaS would be faster to value.
