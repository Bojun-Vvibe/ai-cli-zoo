# arize-phoenix

> Snapshot date: 2026-04. Upstream: <https://github.com/Arize-ai/phoenix>
> License file: <https://github.com/Arize-ai/phoenix/blob/main/LICENSE>
> Pinned: **arize-phoenix v14.14.0** (2026-04-24, PyPI: `arize-phoenix`).

Phoenix is the OSS observability + eval stack purpose-built for LLM
applications: an OpenTelemetry-collector-shaped tracing backend that
understands `gen_ai.*` semantic conventions, a web UI for inspecting
spans / sessions / projects, and a Python eval harness that scores
the captured traces with LLM-as-a-judge or code-based evaluators.
Distinguishing trait against generic OTel back-ends (Jaeger, Tempo)
and against vendor APM (Datadog, New Relic): the trace tree *is* the
agent's reasoning trace — system prompt, tool calls, retrieval hits,
intermediate model outputs are all first-class fields on a span, the
UI lets you click into a span and see exactly the messages the model
saw, and the eval harness can score sampled spans without you wiring
up a separate eval pipeline. Pairs with the OpenInference
instrumentations (one per provider / framework — OpenAI, Anthropic,
LangChain, LlamaIndex, DSPy, CrewAI, smolagents, Pydantic-AI,
Bedrock, Vertex, Mistral, etc.) which auto-emit the right span
shapes against any OTel collector — Phoenix is the collector tuned
to understand them.

## 1. Install footprint

- `pip install arize-phoenix` (Python ≥ 3.9). Pulls FastAPI, Uvicorn,
  SQLAlchemy, Strawberry-GraphQL, OpenTelemetry SDK + proto, pandas,
  numpy, sqlean.py (SQLite with extensions). Heavy install ≈ 250 MB.
- For instrumentation only (no UI / collector), `pip install
  arize-phoenix-otel` is the slim ~10 MB package that wires
  OpenInference instrumentors at an existing OTel endpoint
  (Phoenix or any OTel-compatible backend).
- For evals only, `pip install arize-phoenix-evals` is the
  judge-library subset (no server).
- CLI: `phoenix` — Typer app. Subcommands: `phoenix serve` (boots
  the FastAPI app + GraphQL + OTel HTTP/gRPC ingest on `:6006` by
  default), `phoenix datasets`, `phoenix experiments`. The
  one-liner everyone uses: `python -m phoenix.server.main serve`
  in dev, or the convenience `import phoenix as px; px.launch_app()`
  in a notebook.
- Storage: SQLite by default (`~/.phoenix/`); Postgres via
  `PHOENIX_SQL_DATABASE_URL=postgresql://…` for multi-user / shared
  deployments.
- Docker image `arizephoenix/phoenix:latest` is the canonical
  shared-instance install.

## 2. Repo + version + license

- Repo: <https://github.com/Arize-ai/phoenix>
- Latest release: **arize-phoenix v14.14.0** (2026-04-24); the
  monorepo also publishes `arize-phoenix-client v2.5.0`,
  `arize-phoenix-otel v0.16.0`, `arize-phoenix-evals v3.0.0` on
  independent cadences.
- License: **Elastic License 2.0 (ELv2)** —
  <https://github.com/Arize-ai/phoenix/blob/main/LICENSE>. Free for
  internal use and self-hosting; the ELv2 restriction is "don't
  resell Phoenix-as-a-service" (i.e. don't compete with Arize's
  hosted offering). For the eval / OTel sub-packages, check the
  per-package LICENSE — `arize-phoenix-evals` and
  `arize-phoenix-otel` are independently licensed (Apache-2.0 /
  ELv2 varies by package).
- Default branch: `main`
- Language: Python (server + evals) + TypeScript / React (UI)

## 3. Models supported

Phoenix is provider-agnostic — it ingests OTel spans from *any*
LLM call, regardless of which model produced them. The
OpenInference auto-instrumentors cover OpenAI (Chat + Responses +
Assistants), Anthropic (Messages + tool use), Google Gemini (AI
Studio + Vertex), Mistral, Cohere, Groq, Bedrock, Vertex, LiteLLM,
plus framework-level instrumentors for LangChain, LlamaIndex, DSPy,
CrewAI, Haystack, smolagents, Pydantic-AI, AutoGen, Semantic Kernel,
[guardrails-ai], BeeAI. The judge models for `arize-phoenix-evals`
support the same set (any OpenAI-compatible endpoint, plus native
Anthropic / Gemini / Bedrock clients), so a fully-local pipeline
(local Ollama as system-under-test, local Ollama as judge, local
Phoenix as collector) runs with zero external egress.

## 4. MCP support

**Yes — instrumentation side.** OpenInference ships an MCP
auto-instrumentor (`openinference-instrumentation-mcp`) that wraps
both MCP clients and MCP servers in your process and emits a span
per `tools/list` / `tools/call` / `resources/read` round-trip with
the request payload, response payload, and timing on the span. The
Phoenix UI surfaces these as MCP-typed spans alongside the LLM
spans, so you can see exactly which MCP tool the agent called, with
what args, and what came back — the missing observability layer
that makes "did my agent's MCP server actually return the right
data" debuggable. Phoenix itself is not an MCP server.

## 5. Sub-agent model

None — Phoenix is a passive observer, not an agent. Sub-agent
*tracing* is fully supported: nested agent spans land as a
parent-child tree in the UI, each child's tool calls / LLM calls
fold under it, and aggregate metrics (cost, latency, token count)
roll up automatically. The eval harness runs evaluators in parallel
(`run_evals(dataset, evaluators, concurrency=N)`) but each evaluator
is a stateless LLM-judge or code function — no scheduler, no memory.

## 6. Telemetry stance

**Off** for the OSS server — Phoenix is the telemetry sink, not a
sender; the only egress from `phoenix serve` is to the LLM provider
you configured for evals, plus opt-in cloud sync to
`app.phoenix.arize.com` (set `PHOENIX_API_KEY` + `PHOENIX_COLLECTOR_
ENDPOINT=https://app.phoenix.arize.com`). The web UI binds
`0.0.0.0:6006` by default — use `PHOENIX_HOST=127.0.0.1` for local-
only. The OpenInference instrumentors emit spans only to whatever
OTel endpoint you point them at — by default they target
`http://localhost:6006/v1/traces` (a local Phoenix), but you can
ship to Tempo / Honeycomb / Grafana Cloud / Datadog instead and
still get the LLM-aware span shape. No phone-home from any of the
sub-packages as of v14.14.0.

## 7. Prompt-cache strategy

Not applicable to the server (Phoenix doesn't generate prompts; it
records them). For `arize-phoenix-evals`, judge calls reuse the
provider-side prefix cache automatically; the eval library does not
inject `cache_control` markers. Recorded spans do, however, *show*
cache-hit metadata when the underlying provider returns it
(`gen_ai.usage.cache_read_input_tokens` etc. are first-class span
attributes), so Phoenix is how you measure whether your prompt
caching is actually saving you money in production.

## 8. Hot keybinds (UI / CLI)

The web UI (`http://localhost:6006`) is mouse-driven; useful URL
shortcuts and CLI entry points:

| Action | How |
|--------|-----|
| Boot the server | `phoenix serve` (or `px.launch_app()` in a notebook) |
| Stop the server | `Ctrl+C` in the terminal hosting it |
| Browse traces | `http://localhost:6006/projects/<project>/traces` |
| Filter spans | DSL in the UI search box: `span_kind == 'LLM' and llm.model_name == 'gpt-4o' and latency_ms > 2000` |
| Replay a span as a prompt playground | Click span → "Open in Playground" |
| Export traces to Parquet | `phoenix datasets dump <project>` |
| Run an eval over sampled spans | `phoenix experiments run <experiment.py>` |
| Reset local DB | `rm -rf ~/.phoenix/` |

For programmatic access, the `phoenix.Client()` Python class
wraps the GraphQL API — fetch spans, upsert datasets, register
prompts, log experiment runs, all from a notebook.

## 9. Killer feature, weakness, when to choose

- **Killer:** **OTel as the substrate, LLM-aware spans as the
  product.** Phoenix doesn't ask you to adopt a proprietary SDK —
  you `pip install openinference-instrumentation-openai`, call the
  one-line `OpenAIInstrumentor().instrument()`, and *every*
  `client.chat.completions.create(...)` in your codebase becomes a
  span with `llm.input_messages`, `llm.output_messages`,
  `llm.token_count.prompt`, `llm.token_count.completion`,
  `llm.model_name`, plus tool-call args / responses, all visible in
  the trace tree. Frameworks (LangChain, LlamaIndex, DSPy, CrewAI,
  smolagents, Pydantic-AI) get their own instrumentors that emit
  agent-shaped spans (`AGENT`, `CHAIN`, `RETRIEVER`, `RERANKER`,
  `TOOL`, `LLM`, `EMBEDDING`) so the trace tree mirrors the
  framework's mental model. Same span data drives the eval harness
  — sample N spans from a project, run RAG / agent / hallucination
  evaluators against them, write the scores back as span
  annotations, then filter the UI by score to find the failing
  subset. This loop (instrument → trace → sample → eval → debug →
  fix → re-trace) is the production-LLM iteration cycle, and
  Phoenix is the only OSS tool that ships the whole loop in one
  install.
- **Weakness:** **The web UI assumes you're going to run it.**
  There's no good headless / terminal experience — `phoenix serve`
  + a browser tab is the workflow, which fits poorly into a `tmux`
  + `ssh` operator's day. The dependency surface (FastAPI +
  GraphQL + SQLAlchemy + OTel + pandas) is heavy enough that the
  package is awkward to embed in a downstream library; most
  projects depend on `arize-phoenix-otel` (instrumentation only)
  and run the full Phoenix as a separate Docker container. ELv2
  licensing on the main package matters if your business is
  reselling Phoenix as a hosted service — for internal use it's
  effectively Apache-2.0-equivalent, but the wording trips
  legal-review reflexes the first time it crosses a procurement
  desk.
- **Choose it when:** you are running an LLM application beyond toy
  scale (≥ a few thousand calls/day), have already adopted some
  framework that has an OpenInference instrumentor, and need to
  answer "which prompts are slow / expensive / wrong" without
  building a span-storage layer yourself. Pair with
  [`ragas`](../ragas/) or [`deepeval`](../deepeval/) when you want
  their richer RAG / agent metric catalogs as Phoenix evaluators
  (the eval harness accepts arbitrary callables, so cross-library
  composition is the intended path), and with
  [`promptfoo`](../promptfoo/) for pre-deploy YAML-config eval gates
  on top of the post-deploy production observability Phoenix
  provides. Skip if your stack is one Python script calling
  OpenAI directly and `print(response)` debugging is keeping up —
  Phoenix's value compounds with span volume and framework
  complexity. The catalog's clearest answer to "give me an
  LLM-shaped Datadog I can self-host" without committing to a
  vendor.
