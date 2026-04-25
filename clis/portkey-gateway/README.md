# portkey-gateway

> Snapshot date: 2026-04. Upstream: <https://github.com/Portkey-AI/gateway>

"**A blazing fast AI Gateway with integrated guardrails. Route to
200+ LLMs, 50+ AI Guardrails with 1 fast & friendly API.**" Portkey
Gateway is a single Hono / Cloudflare-Workers-runtime AI gateway
written in TypeScript that fronts ~200 LLM providers (OpenAI,
Anthropic, Bedrock, Vertex, Gemini, Cohere, Mistral, Groq, Together,
Fireworks, DeepSeek, xAI, Perplexity, Ollama, vLLM, llama.cpp, and
the long tail of OpenAI-compatible URLs) behind one OpenAI-shaped
HTTP surface, then layers production primitives — load-balanced
routing, fallback chains, conditional routing on header/body
expressions, exponential-backoff retries, request timeouts,
semantic + simple caching, virtual-key budget caps, per-provider
RPM/TPM limits, and a `guardrails` field that runs PII / toxicity
/ schema / regex / model-graded-rubric checks on input and output —
declared per-request in headers or per-deployment in a config JSON.
The gateway is a stateless single binary you run yourself
(`npx @portkey-ai/gateway` or one Docker container), so it slots
into your VPC with no managed control plane.

## 1. Install footprint

- One-shot local: `npx @portkey-ai/gateway` boots the server on
  `:8787` with no config file (proxy any provider via `x-portkey-
  provider` + `Authorization` headers).
- Docker: `docker run -p 8787:8787 portkeyai/gateway:latest`
  (~150 MB image, Node 20 base).
- Cloudflare Workers: `git clone && npm run deploy` (the gateway
  is built on Hono so it boots unchanged on Workers / Deno / Bun /
  Node runtimes; Workers gives you a sub-50 ms global edge).
- Self-hostable Helm chart in `deploy/helm/` for k8s.
- TypeScript (Hono + provider adapters in
  `src/providers/<vendor>/`); no Python or Go dependency.

## 2. Repo + version + license

- Repo: <https://github.com/Portkey-AI/gateway>
- Latest release: **v1.15.2** (2026 cadence, monthly minors)
- HEAD SHA: `351692fd9236af222168134b416924fae0bdba23`
- License: **MIT** —
  <https://github.com/Portkey-AI/gateway/blob/main/LICENSE>
- Default branch: `main`
- Language: TypeScript

## 3. Models supported

~200 providers via per-vendor adapters: OpenAI (chat / responses /
embeddings / audio / images / batch), Anthropic (messages + tool
use + Bedrock + Vertex routes), Google (Gemini AI Studio + Vertex),
Mistral, Cohere, Groq, Together, Fireworks, Replicate, Hugging Face
Inference, DeepSeek, xAI, Perplexity, Cerebras, SambaNova,
OpenRouter, Lambda Labs, Anyscale, Novita, NVIDIA NIM, Workers AI,
Bedrock (Claude / Titan / Cohere / Llama / Mistral / Jurassic),
Vertex (Gemini + Anthropic + Mistral + Llama + Claude), Azure
OpenAI, plus Ollama / vLLM / LM Studio / llama.cpp / TGI / any
OpenAI-compatible URL via the generic `openai-compatible` provider.
Vendor selection per-request via `x-portkey-provider: anthropic`
header or per-deployment via a config JSON checked into git.

## 4. MCP support

No first-party MCP server or client in the gateway itself — it
operates one layer below MCP, as the LLM transport an MCP-aware
agent calls when the model itself isn't local. Pair with
[`fast-agent`](../fast-agent/) / [`pydantic-ai`](../pydantic-ai/) /
[`opencode`](../opencode/) / [`crush`](../crush/) on the agent side,
pointing them at `http://gateway:8787/v1` with the right virtual
key.

## 5. Sub-agent model

None — gateway layer. Concurrency is request-scoped: each inbound
request runs its declared `strategy` (loadbalance / fallback /
conditional / single) and `targets` graph in parallel as needed,
with per-target retries and timeouts inside one round-trip.

## 6. Telemetry stance

Off in the OSS gateway. The self-hosted binary makes no analytics
calls; egress is exactly the upstream provider you configured plus
inbound HTTP from your clients. Optional `x-portkey-trace-id`
header propagates a correlation ID; optional log hooks (`logger:
{ exporter: "otel", endpoint: "..." }`) fan request / response
metadata to your own OTel collector. The hosted Portkey control
plane (`api.portkey.ai`) is a *separate* commercial product — the
OSS gateway only talks to it if you set `x-portkey-api-key`
explicitly to integrate with their dashboards.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Production routing primitives as request
headers, not as code.** Failover from `gpt-5` to `claude-sonnet-4`
on 5xx is one `strategy: { mode: "fallback", on_status_codes:
[429, 500, 502, 503, 504] }` block in a config JSON; A/B routing
80% to model X and 20% to model Y is `mode: "loadbalance"` with
`weight` fields; conditional routing ("send EU customers to a
Vertex EU endpoint, US customers to OpenAI") is `mode: "conditional"`
with a `query` JMESPath expression over the request metadata; per-
request guardrails (PII, regex, JSON schema, prompt injection,
toxicity, model-graded rubric) attach as a `guardrails` field with
an `on_fail: "deny" | "fallback" | "passthrough"` policy. Same
config file works in dev (one line at a time, headers) and prod
(deployed config with versioned `config-id`s). **Self-hostable
+ MIT** is the differentiator vs. the SaaS-first peers in the
gateway space.

**Weakness.** **The OSS gateway is the data plane, not the control
plane.** Prompt management, dataset curation, eval runs, request
log search, cost dashboards, virtual-key admin UI, and per-team
billing live on the hosted commercial Portkey backend; the OSS
binary records nothing on its own — you fan to your own observability
stack ([`langfuse`](../langfuse/) / [`arize-phoenix`](../arize-phoenix/)
/ [`helicone`](../helicone/) / [`openllmetry`](../openllmetry/) /
plain OTel) via log hooks if you want a UI for traces. Provider
adapter parity drifts (a brand-new OpenAI feature may take a release
to land in every provider's normalized shape); pin the gateway
version in production. Semantic cache is a pre-1.0-quality feature
in the OSS code path — review the threshold and TTL knobs carefully
before enabling on outputs where a near-miss is dangerous (legal,
medical, numerical).

**When to choose.** You have one polyglot codebase that talks to
multiple LLM providers and you want a single self-hostable choke
point for failover, retries, caching, guardrails, and per-key
budgets — without adopting a hosted control plane like
[`helicone`](../helicone/) or paying for managed routing like
OpenRouter. Pair with [`litellm`](../litellm/)'s SDK shim in code
that needs the `openai`-SDK API on the client side; pair with
[`langfuse`](../langfuse/) / [`arize-phoenix`](../arize-phoenix/)
downstream for the trace UI Portkey's OSS gateway intentionally
omits. Skip if you only talk to one provider (no gateway needed),
if you want a managed product with a built-in dashboard
(use the hosted Portkey SaaS or [`helicone`](../helicone/)
instead), or if your routing logic is simpler than "pick a model
based on the request" (a one-line `litellm` call already covers
that case without a separate process to operate).
