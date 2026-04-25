# litellm

> Snapshot date: 2026-04. Upstream: <https://github.com/BerriAI/litellm>
> License file: <https://github.com/BerriAI/litellm/blob/main/LICENSE>
> Pinned: `v1.83.13-nightly` (2026-04-24, the everyday `pip install
> litellm` track). The Python SDK and the proxy server (`litellm
> --config ...`) ship from the same repo and same release cadence;
> pin the exact version because provider adapters and the proxy
> config schema both move fast.

A **single OpenAI-shaped call site that fans out to 100+ providers**
plus an optional self-hostable proxy server (the "AI gateway" shape:
key-rotation, rate-limits, budgets, fallbacks, audit logging,
observability fan-out, virtual keys for downstream apps). Two ways
to use it: (1) `from litellm import completion` is a drop-in for
`openai.chat.completions.create` that translates the same call to
Anthropic / Gemini / Bedrock / Vertex / Cohere / Mistral / Together
/ Groq / Fireworks / OpenRouter / Ollama / vLLM / SageMaker / any
OpenAI-compatible endpoint via the `model="provider/model-id"`
prefix; (2) `litellm --config config.yaml` boots a FastAPI gateway
that downstream apps hit at `http://localhost:4000/v1/chat/completions`
as if it were OpenAI, while you own routing, retries, fallbacks,
key-vault, spend tracking, and per-tenant virtual API keys.

## 1. Install footprint

- **SDK only**: `pip install litellm` (~30 MB; pulls `openai`,
  `httpx`, `tokenizers`, `tiktoken`). Python 3.8+.
- **Proxy server**: `pip install 'litellm[proxy]'` (~150 MB; adds
  `fastapi`, `uvicorn`, `pyjwt`, `prisma`, `redis`, `prometheus-
  client`, `apscheduler`). Or `docker run -p 4000:4000 -v
  $(pwd)/config.yaml:/app/config.yaml ghcr.io/berriai/litellm:main-latest`
  for the everyday container shape.
- **Proxy + Postgres** (multi-tenant, virtual keys, spend tracking):
  add a Postgres URL to the proxy env (`DATABASE_URL=postgresql://
  ...`) and `litellm` runs Prisma migrations on boot.
- **Enterprise build**: `ghcr.io/berriai/litellm-database:main-stable`
  ships with extra middleware (the `enterprise/` directory of the
  repo, which has its own commercial licence — see Licence below).
- Workspace footprint: a `config.yaml` (model list + router rules +
  fallbacks + budgets), optional `.env` (provider keys), optional
  Postgres for the proxy DB. The SDK has no on-disk footprint
  beyond the install.

## 2. License

**MIT for the everyday codebase** — file is `LICENSE` at the repo
root, <https://github.com/BerriAI/litellm/blob/main/LICENSE>. The
`LICENSE` file is explicit about a carveout: **content under the
`enterprise/` directory is licensed under
`enterprise/LICENSE` (commercial)**; everything else (the SDK, the
proxy core, the provider adapters, the routing logic, the spend
tracker, the virtual-key surface) is MIT and free for any use.

Practically: `pip install litellm` and the OSS proxy image give you
the full gateway feature set. The `enterprise/` directory is a
small set of additional middleware (advanced SSO providers, certain
audit-export shapes, prompt-injection guardrails) that sit on top —
not required for the everyday "fan out OpenAI calls to N providers
with fallbacks" use case.

## 3. Models supported

**100+ providers with one call shape**. The `model=` string is the
routing key; the SDK / proxy parses the prefix and dispatches:

- **Frontier**: `openai/gpt-4o`, `openai/o4-mini`, `anthropic/
  claude-opus-4`, `anthropic/claude-sonnet-4`, `gemini/gemini-2.5-
  pro`, `gemini/gemini-2.5-flash`, `vertex_ai/...`, `bedrock/
  anthropic.claude-3-...`, `xai/grok-...`, `deepseek/deepseek-...`.
- **Open-weight via providers**: `groq/llama-3.3-70b`, `together_ai/
  ...`, `fireworks_ai/...`, `replicate/...`, `databricks/...`,
  `cerebras/...`, `mistral/...`, `cohere/...`, `perplexity/...`.
- **Open-weight self-hosted**: `ollama/llama3.1`, `vllm/...`,
  `openllm/...`, `text-completion-openai/...` (any OpenAI-compatible
  URL), `huggingface/...` (TGI / Inference Endpoints).
- **Embeddings**: `litellm.embedding(model="...", input=[...])`
  routes to the same provider matrix.
- **Image, audio (TTS / STT), rerank, moderation**: parallel
  surfaces with the same fan-out (`litellm.image_generation`,
  `litellm.speech`, `litellm.transcription`, `litellm.rerank`,
  `litellm.moderation`).
- **Custom**: subclass `CustomLLM` and register via the
  `custom_provider_map` so a private inference endpoint with a
  bespoke wire format also goes through the gateway.

The translation layer normalises tool-calling, streaming, JSON
mode, vision input, and prompt-caching headers across providers,
so a downstream app written against the OpenAI shape Just Works
when you flip `model=` from `openai/gpt-4o` to `anthropic/
claude-sonnet-4` or `gemini/gemini-2.5-pro`.

## 4. MCP support

**MCP is a first-class transport on the proxy** as of mid-2025 —
the proxy can expose MCP tools to downstream agents and can
*itself* call out to MCP servers as tools that get included in
LLM tool-call rounds. Configure under `mcp_servers:` in
`config.yaml` (each entry is an MCP server URL or stdio command);
the proxy handles tool-list aggregation, name-collision rewrites,
and per-virtual-key MCP allow-lists.

The SDK side has no MCP transport directly — for MCP-aware agent
work, point an MCP-capable CLI ([`opencode`](../opencode/),
[`claude-code`](../claude-code/), [`crush`](../crush/),
[`fast-agent`](../fast-agent/)) at the LiteLLM proxy as its
**LLM endpoint** (set `OPENAI_BASE_URL=http://localhost:4000/v1`),
and that agent's own MCP integrations continue to work — LiteLLM
sits at the LLM-call layer, transparently to the agent's MCP
loop.

## 5. Sub-agent model

None — LiteLLM is an LLM gateway, not an agent runtime. What it
does have, and what often substitutes for "sub-agent routing"
shape, is the **router**: `model_list:` in `config.yaml` defines
N (provider, model, key, weight) tuples, `litellm_settings:` picks
the routing strategy (`simple-shuffle`, `least-busy`, `usage-based-
routing`, `latency-based-routing`, `cost-based-routing`), and
`fallbacks:` defines the per-model failover chain. Combined with
per-key + per-tenant + per-model RPM/TPM limits and per-virtual-key
budgets, the router becomes the "which backend handles this
request" decision layer in front of any agent framework.

## 6. Telemetry stance

**Off by default in the SDK and OSS proxy.** No analytics shipped
with the SDK; the proxy's optional observability fan-out is
explicit and configurable (`callbacks: ["langfuse", "langsmith",
"helicone", "openllmetry", "prometheus", "datadog", "sentry",
"slack", "s3"]` etc.) — set the keys, opt in per-callback. Spend
tracking lives in your Postgres if you wired one; otherwise it is
in-memory.

The proxy's `/metrics` Prometheus endpoint exposes per-model /
per-key request counts + latency histograms + spend, which the
OTel-shaped [`openllmetry`](../openllmetry/) callback can lift
into any OTLP collector for cross-service tracing.

For sensitive workloads, the proxy supports per-team data-egress
controls (`block_team_with_no_db_access: true`,
`enforce_user_role_based_access`) and PII redaction via the
`presidio` callback before requests are logged.

## 7. Prompt-cache strategy

**Two layers, both first-class.**

(1) **Provider-native prompt caching**: when the upstream provider
supports it (Anthropic ephemeral cache breakpoints, OpenAI auto-
cache on long prompts, Gemini implicit + explicit cache, Bedrock
prompt caching), LiteLLM passes the right headers through and
surfaces `cache_creation_input_tokens` / `cache_read_input_tokens`
in the response; you write the cache breakpoints once in the
provider-agnostic shape and the SDK rewrites them per provider.

(2) **Cross-provider response caching** in the proxy: configure a
Redis / S3 / disk cache under `cache:` in `config.yaml`, and the
proxy returns cached responses for identical (or semantically
near, with `mode: semantic`) requests across all providers — the
counterpart shape to [`gptcache`](../gptcache/) but at the gateway
layer instead of the SDK call site, so every downstream app
benefits without per-app code changes.

## 8. Hot keybinds

No TUI; the everyday surfaces are the Python SDK and the proxy's
admin UI at `http://localhost:4000/ui`.

```python
# --- SDK shape ---
from litellm import completion, acompletion
import litellm

# Drop-in for openai.chat.completions.create — works for any provider
resp = completion(
    model="anthropic/claude-sonnet-4",
    messages=[{"role": "user", "content": "Why is the sky blue?"}],
    max_tokens=200,
    temperature=0.2,
)
print(resp.choices[0].message.content)

# Same code, different model — flip the prefix
resp = completion(
    model="gemini/gemini-2.5-flash",
    messages=[{"role": "user", "content": "Why is the sky blue?"}],
)

# Built-in router (in-process, no proxy)
litellm.set_model_list([
    {"model_name": "fast", "litellm_params": {"model": "groq/llama-3.3-70b", "api_key": "..."}},
    {"model_name": "fast", "litellm_params": {"model": "together_ai/...", "api_key": "..."}},
])
resp = completion(model="fast", messages=[...])  # round-robins across both backends
```

```yaml
# --- Proxy config.yaml shape ---
model_list:
  - model_name: gpt-4o            # alias the downstream apps see
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY
      rpm: 600
  - model_name: gpt-4o            # second backend, same alias = router
    litellm_params:
      model: azure/gpt-4o-deployment
      api_base: os.environ/AZURE_API_BASE
      api_key:  os.environ/AZURE_API_KEY
litellm_settings:
  drop_params: true
  routing_strategy: latency-based-routing
  fallbacks:
    - gpt-4o: ["anthropic/claude-sonnet-4"]   # if both gpt-4o backends fail
  cache:
    type: redis
    host: os.environ/REDIS_HOST
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY   # gates the admin UI + virtual-key creation
  database_url: os.environ/DATABASE_URL        # Postgres for spend + virtual keys
```

```bash
litellm --config config.yaml --port 4000
# Downstream apps now point OPENAI_BASE_URL at http://localhost:4000/v1
# and use a virtual API key minted in the admin UI
```

## 9. Killer feature, weakness, when to choose

**Killer feature.** **One OpenAI-shaped call site that survives every
vendor swap, plus a self-hostable gateway you actually own**. The
SDK kills the "rewrite the call site to migrate from OpenAI to
Anthropic" tax — `model="anthropic/claude-sonnet-4"` is the entire
diff. The proxy kills the "every internal app embeds API keys for
every provider" anti-pattern — apps see one OpenAI-shaped URL and
a virtual key, you own the actual provider keys + the routing +
the budgets + the fallbacks + the audit log + the per-tenant cost
attribution. The fact that the SDK and the proxy come from the
same repo and the same call shape means you graduate "in-process
SDK only" → "shared proxy for the team" without rewriting calling
code; the model alias in `config.yaml` is the only thing that
changes.

**Weakness.** **Translation-layer drift at the edges**. When a
provider ships a new feature (cache breakpoints, structured
output, parallel tool calls, vision-with-detail-control), there is
a window between the provider release and LiteLLM's adapter
catching up; in that window the workaround is `extra_body={...}`
or `headers={...}` to pass through provider-specific shape, which
defeats the point. Streaming + tool-calling normalisation across
providers occasionally surfaces one-off bugs that only repro on
specific (provider, model, tool-shape) combinations — pin the
version and test the call-site shape against the actual providers
you route to. The MIT-with-`enterprise/`-carveout licence is
clear but worth re-reading before redistributing the Docker image
in a commercial product. Default `master_key` distribution + the
admin UI are powerful — keep the proxy off the public internet
and rotate keys on a calendar.

**When to choose.**

- You want **one call site that works against every LLM provider**
  in either the SDK shape or the gateway shape, with the same
  config moving cleanly from one to the other.
- You want **provider failover + retries + cost-aware routing**
  without writing the orchestration code yourself.
- You're standing up a **shared LLM gateway for a team** —
  virtual keys, per-key RPM/TPM limits, per-team budgets, spend
  attribution, audit logging, callback fan-out to your existing
  observability stack.
- You want **prompt caching + response caching as a gateway
  feature**, transparent to downstream apps.
- You want **MCP at the gateway** — agents talk to LiteLLM, LiteLLM
  fans tool calls out to your MCP servers with per-key allow-lists.

**When not to choose.** You only call **one provider in one app**
— the OpenAI / Anthropic SDK directly is fewer dependencies, no
gateway to operate. You want **a managed gateway with someone
else's SLA** — Helicone / Portkey / OpenRouter are commercial-
hosted alternatives. You need **on-prem-only with zero outbound
allow-list flexibility** and the team is already on a single
vendor's gateway product (Bedrock / Vertex front door) — that
front door does most of what LiteLLM's proxy does, scoped to one
vendor. Your real ask is **agent orchestration** (planning, tool
loops, sub-agents) — LiteLLM is the LLM-call layer underneath an
agent framework, not the agent itself; pair with one of the agent
runtimes in this catalog.
