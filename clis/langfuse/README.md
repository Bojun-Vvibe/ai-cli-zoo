# langfuse

> Snapshot date: 2026-04. Upstream: <https://github.com/langfuse/langfuse>
> License file: <https://github.com/langfuse/langfuse/blob/main/LICENSE>
> Pinned: `v3.170.0` (2026-04-23, server). The Langfuse server is the
> source of truth; the SDKs (`pip install langfuse`,
> `npm install langfuse`, plus a Java client) and the integrations
> matrix (LangChain, LlamaIndex, OpenAI SDK callback, LiteLLM
> `callbacks: ["langfuse"]`, OpenTelemetry exporter) track minor
> versions in step.

A **self-hostable LLM engineering platform** — traces + evals +
prompt management + datasets + a playground, all behind one Postgres
+ Clickhouse + Redis stack you `docker compose up`. The contrast
point against [`arize-phoenix`](../arize-phoenix/) is "bundled
product UI vs. OTel-native span store": Phoenix is closer to a
notebook-first OTel viewer with first-class evals, Langfuse is
closer to a hosted-grade product (multi-project workspaces, RBAC,
SSO, prompt versioning with deploy labels, dataset → experiment →
comparison loops) you can run on your own boxes.

## 1. Install footprint

- **Server (everyday shape)**: `git clone https://github.com/langfuse/langfuse && cd langfuse && docker compose up`
  brings up the web UI on `:3000`, Postgres on `:5432`, Clickhouse
  on `:8123`, Redis on `:6379`, and a MinIO blob store on `:9090`.
  ~3 GB of images, ~500 MB RSS at idle.
- **SDKs**:
  - Python: `pip install langfuse` (~25 MB, pulls `httpx` +
    `pydantic`). The `@observe()` decorator turns any function into
    a span; `langfuse.openai` is a drop-in for `openai` that
    auto-traces every chat / responses / embeddings call.
  - Node / TS: `npm install langfuse` (Node 18+).
  - Java: `com.langfuse:langfuse-java` on Maven Central.
- **Integrations** (no extra install beyond the SDK): LangChain
  callback, LlamaIndex callback, [`litellm`](../litellm/) proxy
  callback (`callbacks: ["langfuse"]`), [`dspy`](../dspy/) tracker,
  Vercel AI SDK, OpenTelemetry OTLP exporter (any OTel-instrumented
  app pipes to `/api/public/otel/v1/traces`).
- **Managed**: Langfuse Cloud (US + EU regions) for teams that
  don't want to operate Postgres + Clickhouse; same SDK + same UI,
  different base URL.

## 2. License

MIT for the core repo (file is `LICENSE` at the repo root,
<https://github.com/langfuse/langfuse/blob/main/LICENSE>) — but
**read the header**: portions under `ee/`, `web/src/ee/`, and
`worker/src/ee/` are licensed separately under `ee/LICENSE`
(commercial enterprise terms covering SSO connectors beyond Google
OAuth, fine-grained RBAC, and a few audit-log features). The
"everyday self-host" path (traces, evals, prompt management,
datasets, playground, single-tenant SSO via Google) stays MIT;
SAML / OIDC enterprise SSO + per-project RBAC tiers + advanced
audit need a Langfuse Enterprise key.

## 3. Models supported

Langfuse is a **trace store + evaluator + prompt CMS** — the model
axis is "what did your app call". Any LLM provider works because
the SDK records the request + response + metadata; the bundled
LLM-as-judge evaluators (and the Playground) need at least one
model configured per project, and the supported set is the union
of OpenAI / Anthropic / Bedrock / Azure OpenAI / Vertex / Gemini /
HuggingFace Inference / [`ollama`](../ollama/) / any OpenAI-
compatible URL via the "OpenAI-compatible" provider type. Embedding
calls trace the same way as completion calls.

For evals specifically, Langfuse ships a curated catalog
(hallucination, helpfulness, conciseness, relevance, toxicity,
context-relevance, context-recall, plus user-defined templates) and
you can pick a different judge model per evaluator (cheap router
for the high-volume scorer, smart synthesis for the gating
evaluator).

## 4. MCP support

No first-party MCP server in the core repo as of v3.170. Common
shape is the other direction: agents that already speak MCP
(`opencode`, `claude-code`) trace through Langfuse via OpenTelemetry
or the OpenAI SDK shim, and prompts pulled from Langfuse Prompt
Management (`langfuse.get_prompt("agent-system", label="production")`)
feed those agents at runtime. Community MCP wrappers exist but pin
to a specific SDK version — verify before adoption.

## 5. Sub-agent model

None — Langfuse is observability + governance for agents that live
elsewhere. The trace model is hierarchical (trace → span → span),
so a multi-agent system ([`crewai`](../crewai/),
[`smolagents`](../smolagents/), [`pydantic-ai`](../pydantic-ai/))
shows up as nested spans with parent/child links and
per-span model + latency + cost + token counts; the UI's "trace
detail" view is the natural debug surface for "which sub-agent
spent the tokens".

## 6. Telemetry stance

Self-hosted Langfuse has **no outbound telemetry** — the server
phones home only for the version-check banner (disable via
`TELEMETRY_ENABLED=false`). All traces, prompts, datasets, and
eval results live in your Postgres + Clickhouse + MinIO. Langfuse
Cloud is an opt-in managed product with its own data processing
agreement; you choose at install time.

For sensitive prompts, the SDK's `mask` callback runs client-side
before the trace leaves the process — redact PII / credentials
before they hit the network, with no server-side scrubbing
dependency.

## 7. Prompt-cache strategy

Not a cache — but the **Prompt Management** module is the upstream
of any cache: prompts have versions, deploy labels (`production`,
`staging`, `latest`), and a content hash, and the SDK caches
fetched prompts in-process by label with a configurable TTL
(`langfuse.get_prompt("foo", cache_ttl_seconds=60)`). Pair with
[`gptcache`](../gptcache/) or LiteLLM proxy caching for response-
side caching; Langfuse owns the prompt-template-side caching.

## 8. Hot keybinds

No TUI; the everyday surfaces are the web UI (`http://localhost:3000`,
sidebar: Tracing → Sessions → Prompts → Datasets → Evaluations →
Playground → Settings) and the SDK.

```python
from langfuse import Langfuse, observe
from langfuse.openai import openai  # drop-in shim — auto-traces

lf = Langfuse(
    public_key="pk-lf-...", secret_key="sk-lf-...",
    host="http://localhost:3000",
)

# 1. Pull a versioned prompt with a deploy label
prompt = lf.get_prompt("agent-system", label="production")
system = prompt.compile(user_name="Alice", role="staff")

# 2. Make a traced LLM call (the openai shim records it as a span)
@observe()  # the wrapping function becomes the parent trace
def answer(question: str) -> str:
    resp = openai.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "system", "content": system},
                  {"role": "user", "content": question}],
    )
    return resp.choices[0].message.content

answer("What's our refund window?")

# 3. Score the trace (could also be done via online eval rules in the UI)
lf.score(trace_id=lf.get_current_trace_id(),
         name="user-thumbs", value=1)
```

Online evaluation rules in the UI run the configured evaluators
(LLM-as-judge or your custom Python) against every new trace
matching a filter, and write the score back to the trace — no
batch job to schedule.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Traces + evals + prompt management +
datasets + playground in one self-hostable MIT install** — the
"stitch Phoenix + PromptLayer + Argilla + a homegrown prompt CMS"
stack collapses to one `docker compose up`. Prompt versioning
with deploy labels (`production` / `staging` / canary) is the
load-bearing piece: prod code does
`lf.get_prompt("foo", label="production")`, prompt engineers ship
new versions in the UI, no app deploy needed; rollback is one
click. Online eval rules score every trace automatically and
write the score back to the span, so the regression curve for
"hallucination rate over the last 7 days" is a UI chart, not a
notebook script.

**Weakness.** **Operational footprint.** Postgres + Clickhouse +
Redis + MinIO is a real stack to run; the "five-minute SQLite"
shape doesn't exist (Clickhouse is load-bearing for the high-
cardinality span queries). The MIT / EE split needs a careful
read before assuming "self-host = full feature set" — SAML SSO,
fine-grained RBAC, and some audit features are EE-only. Schema
migrations across major versions occasionally need a planned
window; pin the server image and read the migration notes.

**When to choose.**

- You want **prompt management with deploy labels** as the
  contract between prompt engineers and prod code, not a
  hard-coded string in the repo.
- You want **traces + evals in one UI** and the operational cost
  of running Postgres + Clickhouse is acceptable.
- You're already piping calls through [`litellm`](../litellm/) —
  `callbacks: ["langfuse"]` is one config line and you get the
  full trace + cost view across every provider in your routing
  pool.
- You need **dataset → experiment → comparison** loops to A/B
  prompts or models with statistical comparison built into the UI,
  not a Jupyter notebook.

**When not to choose.** You want **OTel-native, no-product-UI**
trace storage — pipe to Tempo / Jaeger / Honeycomb / Datadog
directly via OTLP. You want a **notebook-first eval workflow**
with no separate server — [`arize-phoenix`](../arize-phoenix/)
runs in-process and is friendlier in a notebook. You want a
**pure eval framework** with no observability product attached —
[`deepeval`](../deepeval/) is pytest-shaped and ships nothing to
operate. You want **bundled UI under straight Apache-2.0** with no
EE carve-outs — [`opik`](../opik/) is the closer match.
