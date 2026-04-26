# laminar

> Snapshot date: 2026-04. Upstream: <https://github.com/lmnr-ai/lmnr>

An **open-source observability + evaluation platform purpose-built for AI
agents**. Self-hostable Postgres + Clickhouse + Rust ingestion stack with a
TypeScript console; instruments your agent code with one decorator
(`@observe`) and gets you span-level traces (LLM call → tool call → child
agent → tool call → LLM call → ...) plus a built-in evaluation harness, a
prompt playground, a labelling queue, and a browser-agent recorder — all
behind one self-hosted endpoint.

The pitch is "Datadog for agents, but the open-source one you actually
self-host instead of paying per-span." YC S24 alumni; Apache-2.0 across
the whole repo (no `enterprise/` carve-out).

## 1. Install footprint

- **Self-host:** `docker compose up` against the repo's `docker-compose.yml`
  brings up Postgres + Clickhouse + the Rust ingestion service + the
  Next.js console at `localhost:5667`.
- **Python SDK:** `pip install lmnr` then `from lmnr import Laminar,
  observe; Laminar.initialize(project_api_key="...")`.
- **TypeScript SDK:** `npm install @lmnr-ai/lmnr`.
- **CLI:** `npx lmnr` for project init, eval runs, dataset import.
- ~150 MB Docker stack at rest (Clickhouse dominates); the SDK is
  ~10 MB pip.
- Hosted SaaS available at `lmnr.ai` for teams that don't want to operate
  Clickhouse — same SDK, same console, different endpoint.

## 2. Repo, version, license

- Repo: <https://github.com/lmnr-ai/lmnr>
- Version checked: **`v0.1.43`** (released 2026-04-14); `main` at
  `94863c3` (2026-04-14). Active release cadence — multiple tags per
  month.
- License: **Apache-2.0**. License file at the repo root: `LICENSE.md`.
- PyPI: `lmnr`. npm: `@lmnr-ai/lmnr`.

## 3. What it actually does

Five surfaces, one backend:

1. **Tracing.** `@observe` decorator wraps a function and emits an
   OpenTelemetry span; nested `@observe` calls produce parent / child
   span hierarchies. Auto-instrumentation for `openai`, `anthropic`,
   `litellm`, `langchain`, `llamaindex`, `crewai`,
   `openai-agents`, `pydantic-ai`, and a dozen others — install the
   SDK, set the env var, every model call shows up in the console
   without touching the code.
2. **Evals.** `lmnr eval run my_eval.py` executes a Python file that
   declares a dataset + an `executor(input) -> output` + an
   `evaluator(output, target) -> score`; results stream into the
   console with diff vs previous run, so PR-time regression checks
   are a one-liner in CI.
3. **Datasets.** Spans → datasets in two clicks (the "curate from
   traces" workflow): filter spans by tag / score / date, hit
   "create dataset", get a versioned dataset usable as the input to
   an eval. The curation loop closes — production traces become
   evals become regression gates.
4. **Playground.** Side-by-side prompt comparison across providers
   (OpenAI / Anthropic / Gemini / Bedrock / Groq / DeepSeek / Ollama
   / any OpenAI-compatible) with cost + latency + token counts, plus
   "promote to production" that publishes the prompt as a versioned
   asset the SDK fetches at runtime.
5. **Browser-agent recorder.** Optional companion that records
   browser-agent runs (Playwright + screenshots + DOM snapshots) and
   plays them back inside the trace UI, so a failing
   [`browser-use`](../browser-use/) / [`skyvern`](../skyvern/) / hand-
   rolled Playwright agent run is reproducible without re-driving the
   browser.

## 4. MCP support

Yes — the auto-instrumentation hooks MCP client traffic when the host
agent uses an MCP-aware framework (OpenAI Agents SDK, pydantic-ai,
fast-agent), so MCP tool calls show up as first-class spans alongside
HTTP and LLM spans. Laminar itself does not expose an MCP server.

## 5. Sub-agent model

None — Laminar observes other people's agents, it does not run them.
Multi-agent traces from frameworks that *do* spawn sub-agents
([`crewai`](../crewai/), [`agency-swarm`](../agency-swarm/),
[`openai-agents-python`](../openai-agents-python/),
[`agentscope`](../agentscope/)) render as parent / child span trees.

## 6. Telemetry stance

**Self-hosted:** off. The Docker stack ingests only what your SDK
sends, stores it locally, calls nothing home. The console is served
from the same stack.

**Hosted (lmnr.ai):** the SDK ships spans to `api.lmnr.ai` — that is
the SaaS product. Pick self-hosted if data residency matters.

The SDK can redact span attributes by key pattern before egress
(`Laminar.initialize(mask_input_keys=["password", "ssn"])`), so
"trace everything except secrets" is a one-line config rather than a
custom span processor.

## 7. Token / context strategy

The SDK does not touch your prompts. Trace payload size is bounded by
sampling (`Laminar.initialize(sample_rate=0.1)`) and by per-span
attribute limits (16 KB default per attribute, configurable). Long
prompts / responses are stored truncated with a pointer to the full
blob in object storage (S3 / MinIO when self-hosted) so the console
loads fast on million-span projects.

## 8. Hot keybinds

N/A — the console is web UI, not TUI. `lmnr` CLI surface is
`lmnr init`, `lmnr eval run`, `lmnr dataset import`,
`lmnr prompt push`.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Curate-from-traces.** Production spans → tagged
dataset → eval → CI gate is a single workflow inside one tool, no
glue between an observability product and an eval product. The
closest sibling [`langfuse`](../langfuse/) splits this across
"traces" + "datasets" + "evaluations" with manual stitching;
Laminar treats them as one loop and the UI affords it. Apache-2.0
across the whole repo (no AGPL server / no EE carve-out) makes it
deployable inside any commercial product.

**Weakness.** Clickhouse in the stack is a real ops dependency —
self-hosting is "spin up Postgres + Clickhouse + Rust service +
Next.js console" not a single binary, so the bar to "I just want to
look at one trace" is higher than [`logfire`](../logfire/) (single
hosted endpoint) or [`openllmetry`](../openllmetry/) (just OTel
SDKs you point anywhere). Browser-agent recorder is the newest
surface and the docs lag the code. Smaller community than
Langfuse / Phoenix — fewer pre-built integrations for niche
frameworks (write your own auto-instrumentation if your stack is
exotic).

**When to choose.**
- You ship an **agent product** (multi-step tool-using LLM apps) and
  need **traces + evals + datasets** in one tool, not three.
- You need **self-hosted Apache-2.0** observability with no per-span
  pricing and no EE carve-out — for regulated industries or for
  embedding the observability layer inside your own product.
- You want **production-trace → eval-dataset → regression-gate** as
  one curated workflow, not a stitched-together pipeline.
- You run **browser agents** and need step-level replay of failed
  runs.

**When to skip.**
- You want **the lightest possible OTel surface** with no UI you
  operate yourself — use [`openllmetry`](../openllmetry/) +
  whatever OTel backend you already have.
- You want a **hosted dev-friendly** trace viewer with one-line setup
  and no Clickhouse → [`logfire`](../logfire/) (Pydantic).
- You want **the most mature OSS LLM observability** with the biggest
  ecosystem → [`langfuse`](../langfuse/).
- You want **eval-only**, no observability → [`promptfoo`](../promptfoo/),
  [`deepeval`](../deepeval/), [`ragas`](../ragas/).
- You want **adversarial testing** specifically →
  [`deepteam`](../deepteam/) or [`garak`](../garak/).

## 10. Compared to neighbors in the catalog

| Tool | Trace + eval in one | Self-host shape | License | Curate-from-traces |
|------|---------------------|-----------------|---------|--------------------|
| laminar | Yes | Postgres + Clickhouse + Rust + Next.js | Apache-2.0 (whole repo) | Yes (one-click) |
| [langfuse](../langfuse/) | Yes (split surfaces) | Postgres + Clickhouse + Node | MIT (server has EE carve-out) | Yes (manual) |
| [arize-phoenix](../arize-phoenix/) | Trace + eval | Single Python service | Elastic-2.0 | Limited |
| [logfire](../logfire/) | Trace only | Hosted only (OSS SDK) | MIT (SDK) / proprietary (backend) | No |
| [openllmetry](../openllmetry/) | Trace only (SDK) | BYO OTel backend | Apache-2.0 | No |
| [helicone](../helicone/) | Trace + cache | Postgres + Clickhouse + Worker | Apache-2.0 | No |
| [langtrace](../langtrace/) | Trace + eval | Postgres + Clickhouse | AGPL-3.0 (server) | Limited |
| [pezzo](../pezzo/) | Prompt CMS + thin trace | Postgres + Clickhouse + Redis | Apache-2.0 | No |

Decision shortcut:

- "Trace **and** eval **and** dataset curation in one Apache-2.0
  self-host" → `laminar`.
- "Most mature OSS LLM observability, biggest ecosystem" →
  `langfuse`.
- "Single hosted endpoint, no ops" → `logfire`.
- "Just OTel SDKs into my existing backend" → `openllmetry`.
- "Cache + observability at the gateway hop" → `helicone`.
- "Prompt CMS first, observability second" → `pezzo`.
