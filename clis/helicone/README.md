# helicone

> Snapshot date: 2026-04. Upstream: <https://github.com/Helicone/helicone>
> License file: <https://github.com/Helicone/helicone/blob/main/LICENSE>
> Pinned: `v2025.08.21-1` (2025-08-21, last tagged release; the
> hosted product ships continuously off `main`, the OSS image lags
> the cloud by weeks). The Helicone proxy is the source of truth;
> the SDKs (`pip install helicone`, `npm install @helicone/helicone`)
> are thin clients that mostly just configure headers, since the
> "one-line integration" path is a base-URL swap on the OpenAI SDK.

An **open-source LLM observability platform** built around an HTTP
proxy that sits between your app and the LLM provider — change the
OpenAI base URL from `https://api.openai.com/v1` to
`https://oai.helicone.ai/v1` (or self-host and point at your own
URL), and every request + response + cost + latency lands in the
Helicone dashboard with no code change beyond the URL + an
`Helicone-Auth` header. The contrast point against
[`langfuse`](../langfuse/) is "proxy-first vs. SDK-first":
Langfuse traces by wrapping the SDK call (`@observe()` decorator,
LangChain callback), Helicone traces by sitting on the wire so
*every* call from *every* language hits the same recorder without
per-language SDK glue.

## 1. Install footprint

- **Cloud (default path)**: change base URL + add header. No
  install. Free tier covers 100K requests/month.
- **Self-host (everyday shape)**: `git clone https://github.com/Helicone/helicone && cd helicone/docker && docker compose up`
  brings up the worker (Cloudflare-Workers-compatible JS via
  `wrangler dev`), the web UI, Clickhouse for high-cardinality
  request storage, Postgres for app metadata, MinIO for request /
  response body archival, and Kafka for the ingest queue. ~4 GB of
  images, expect ~1 GB RSS at idle — heavier than Langfuse because
  the request-body archive path is built in.
- **SDKs (optional, for advanced features)**:
  - Python: `pip install helicone` — exposes
    `from helicone.openai_proxy import openai` for the auto-config
    path, and `helicone_async` for batched uploads.
  - Node / TS: `npm install @helicone/helicone`.
  - **The everyday integration is just headers, no SDK**:
    `OpenAI(base_url="https://oai.helicone.ai/v1", default_headers={"Helicone-Auth": "Bearer sk-helicone-..."})`
    works from any language with an OpenAI client.
- **Gateway mode** (not the proxy): `helicone-gateway` is a newer
  Rust binary that adds routing + load balancing on top of the
  observability layer — closer in shape to LiteLLM proxy than to
  the legacy JS Worker.

## 2. License

Apache-2.0 for the everyday codebase (file is `LICENSE` at the
repo root,
<https://github.com/Helicone/helicone/blob/main/LICENSE>). The
worker, web UI, SDKs, and gateway are Apache-2.0. Helicone Cloud
is the commercial managed offering; the OSS image has no feature
gating beyond "you operate Clickhouse + Kafka + Postgres yourself".

## 3. Models supported

Any provider with an OpenAI-compatible HTTP surface routes through
the proxy by base-URL swap: OpenAI, Azure OpenAI, Anthropic (via
`https://anthropic.helicone.ai/v1`), Bedrock, Vertex / Gemini,
Groq, Together, Fireworks, OpenRouter, DeepSeek, xAI, Cohere,
Mistral, [`ollama`](../ollama/), [`vllm`](../vllm/), any
OpenAI-compatible URL. Each provider has a documented Helicone
base URL that does the auth pass-through and translation. Cost
attribution is computed from the model + token counts using a
maintained price table — accurate within rounding for the major
providers, "best-effort" for niche or self-hosted models where you
can override via per-request `Helicone-Cost-Override` header.

## 4. MCP support

No first-party MCP server in the OSS repo as of `v2025.08.21-1`.
The proxy logs MCP-related tool calls if your agent puts them in
the OpenAI request payload (`tool_calls` array), so an MCP-aware
agent ([`opencode`](../opencode/), [`claude-code`](../claude-code/),
[`crush`](../crush/)) routed through Helicone gets per-tool-call
visibility for free; first-class MCP server hosting is a Cloud
roadmap item, not OSS.

## 5. Sub-agent model

None — Helicone is observability + routing for the LLM call layer.
Sub-agent topology shows up as nested requests linked by
`Helicone-Session-Id` + `Helicone-Session-Path` headers (e.g.
`/planner/researcher/synthesis`), which the UI reconstructs as a
tree view per session — the multi-agent debug equivalent of a
Jaeger trace, but for LLM calls only.

## 6. Telemetry stance

Self-hosted Helicone keeps everything in your Clickhouse +
Postgres + MinIO; no outbound telemetry from the worker beyond the
upstream LLM provider call itself. Cloud is opt-in.

The proxy logs **request and response bodies by default** — this
is the load-bearing feature for cost attribution + replay, and
also the biggest privacy surface. For sensitive payloads, the
`Helicone-Omit-Request-Body` and `Helicone-Omit-Response-Body`
headers drop body capture per request, and the `Helicone-Property-*`
header family lets you tag without storing the prompt content.
For PII, the `Helicone-PII-Redaction-Enabled` header runs
server-side regex + LLM redaction (Cloud) or you wire a redaction
hook in the worker (self-host).

## 7. Prompt-cache strategy

**Built-in semantic cache** at the proxy layer — set
`Helicone-Cache-Enabled: true` and `Helicone-Cache-Bucket-Max-Size: 10`
headers, the worker hashes the request (or embeds it for semantic
mode via `Helicone-Cache-Similarity-Threshold`), looks up the
bucket in Cloudflare KV (Cloud) / Redis (self-host), and returns
the cached response when it hits. Cache key namespacing is
per-`Helicone-User-Id` + per-prompt by default, with `Helicone-
Cache-Seed` for cache busting. The contrast point against
[`gptcache`](../gptcache/) is "where the cache lives": GPTCache is
SDK-side (cache before the network), Helicone is gateway-side
(cache after the SDK leaves the process) — the gateway shape wins
when you have many call sites in many languages and want one
cache; the SDK shape wins when you want zero-network-hop cache
hits.

## 8. Hot keybinds

No TUI; the everyday surfaces are the web UI (`http://localhost:3000`,
sidebar: Requests → Sessions → Properties → Users → Prompts →
Experiments → Settings) and the proxy headers themselves.

```python
from openai import OpenAI

# 1. The entire integration: base URL + auth header
client = OpenAI(
    base_url="https://oai.helicone.ai/v1",  # or your self-hosted URL
    api_key="sk-openai-...",
    default_headers={
        "Helicone-Auth": "Bearer sk-helicone-...",
        "Helicone-User-Id": "user-42",                  # per-user cost
        "Helicone-Property-Feature": "rag-answer",      # arbitrary tag
        "Helicone-Session-Id": "session-abc",           # multi-agent tree
        "Helicone-Session-Path": "/planner/researcher", # parent path
        "Helicone-Cache-Enabled": "true",               # turn on cache
        "Helicone-Prompt-Id": "rag-system-v3",          # prompt versioning
    },
)

# 2. Make the call — Helicone records request, response, cost, latency
resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Summarise the contract."}],
)
print(resp.choices[0].message.content)
```

Headers are the API. The full set covers caching, rate limits per
user / per key, retries, fallback (`Helicone-Fallback-Models`),
prompt versioning, and per-request cost overrides.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **One-line, language-agnostic LLM
observability via base-URL swap** — no SDK to install, no
decorator to add, no callback to register. Polyglot stacks (Python
service + TS frontend + Go batch worker + Ruby admin) all hit the
same recorder by changing one URL each, and the proxy adds
session reconstruction (multi-agent trees), per-user cost
attribution, semantic caching, fallback routing, and prompt
versioning as header-only opt-ins. The Apache-2.0 self-host has
no feature gating against the Cloud product beyond "you run
Clickhouse + Kafka yourself".

**Weakness.** **The proxy is on the critical path.** Every LLM
request goes through Helicone-the-network-hop, so a self-hosted
worker that goes down takes your LLM calls with it (Cloudflare
Workers + Cloudflare KV is the Cloud topology and is fast +
durable; self-hosting on a single Docker host is *not*). Mitigation
is to run the worker behind a load balancer with a
`Helicone-Auth-Strategy: async` header so failures fall through to
the upstream provider and the trace lands later. The OSS Docker
compose stack is heavier than Langfuse's (Kafka + Clickhouse + worker
+ web + Postgres + MinIO) — fine for a team install, overkill for
a laptop. The OSS image release cadence lags the Cloud product by
weeks; the rolling `:latest` Docker tag is closer to current.

**When to choose.**

- You have a **polyglot stack** and want LLM observability without
  per-language SDK glue — the base-URL swap works from any HTTP
  client.
- You want **a real semantic cache at the gateway layer** that
  applies to every call site without per-call-site code, plus
  per-user / per-feature cost attribution from headers.
- You need **fallback routing + retries on provider errors** as
  header-driven policy without standing up a separate gateway like
  [`litellm`](../litellm/) (though they compose — Helicone in front
  of LiteLLM gives you observability + routing both).
- You want **Apache-2.0 OSS with no MIT-vs-EE carve-outs** —
  contrast with [`langfuse`](../langfuse/)'s `ee/` directory.

**When not to choose.** You want **a real prompt CMS + dataset +
experiment + LLM-as-judge eval workflow in one UI** —
[`langfuse`](../langfuse/) and [`opik`](../opik/) are denser
products on those axes. You want **OTel-native span ingest into
Tempo / Jaeger / Datadog** without a bespoke proxy — Helicone is
its own data plane, not OTLP. You don't want **a network hop in
front of your LLM calls** — keep tracing SDK-side via
[`langfuse`](../langfuse/) or [`openllmetry`](../openllmetry/).
You want a **routing-first AI gateway** with observability
secondary — [`litellm`](../litellm/)'s proxy is the better fit.
