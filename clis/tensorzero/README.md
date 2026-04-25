# tensorzero

> Snapshot date: 2026-04. Upstream: <https://github.com/tensorzero/tensorzero>

**Open-source LLMOps platform: gateway + observability + eval +
optimization in one Rust binary.** TensorZero is a single
self-hostable service that sits between your application and N
LLM providers; it exposes an OpenAI-shaped HTTP API, persists
every inference + downstream feedback signal in ClickHouse, and
closes the loop by *using* that data to fine-tune, A/B test, and
auto-route across models — the gateway, the trace store, the
eval harness, and the optimizer are one binary, one schema, one
config file.

## Repo + version + license

- Repo: <https://github.com/tensorzero/tensorzero>
- Latest release: **`2026.4.1`** (2026-04-24)
- HEAD on `main`: `7a6159a`
- License: **Apache-2.0** —
  <https://github.com/tensorzero/tensorzero/blob/main/LICENSE>
- License path in repo: `LICENSE`
- License SPDX: `Apache-2.0`
- Default branch: `main`
- Language: Rust (gateway) + Python / TypeScript clients

## Install

```bash
# Run the gateway via Docker Compose (gateway + ClickHouse + UI)
curl -LsSf https://tensorzero.com/install.sh | sh
docker compose up -d

# Or Python client against your own gateway:
pip install tensorzero
# Then in code:
#   from tensorzero import TensorZeroGateway
#   client = TensorZeroGateway.build_http(gateway_url="http://localhost:3000")
#   r = client.inference(function_name="generate_haiku", input={...})

# UI at http://localhost:4000 — observability, evals, experiments
```

## Niche

The "**self-hosted gateway that doubles as an experiment platform**"
lane. Overlaps with [`litellm`](../litellm/) on the
OpenAI-compatible-proxy axis, with [`langfuse`](../langfuse/) /
[`helicone`](../helicone/) /
[`openllmetry`](../openllmetry/) on the observability axis, and
with [`promptfoo`](../promptfoo/) / [`deepeval`](../deepeval/) on
the eval axis — but ships a single Rust binary that does all
three behind one config (`tensorzero.toml`) and one trace schema
(ClickHouse), so the eval framework can query the live
production traces directly without an export pipeline.

## Why it matters

- **One binary, one schema.** The gateway routes inference, the
  same process writes the trace, the same trace becomes the eval
  dataset, the same dataset feeds the fine-tuner — no Kafka, no
  data warehouse export, no "wire Langfuse to S3 to Snowflake to
  the eval CI". ClickHouse is the substrate; SQL is the query
  surface.
- **Functions, not prompts.** You declare a `[functions.X]` in
  TOML with input schema, output schema, and N candidate
  variants (different prompts × different models). The gateway
  picks a variant per request (weighted, A/B, or contextual
  bandit), records the choice + outcome, and the built-in
  optimizer learns which variant wins per slice — closed-loop
  prompt + model selection without a hand-rolled experiment rig.
- **Native fine-tuning loop.** `tensorzero export` produces an
  OpenAI / Anthropic / Together / Fireworks-shaped fine-tune
  dataset filtered by feedback metric (e.g. "all traces where
  the user upvoted"); fine-tunes registered back as a new
  variant; the bandit starts routing traffic to it
  automatically.
- **Provider matrix flows through one OpenAI-shaped URL.**
  Anthropic, OpenAI, Gemini (AI Studio + Vertex), Bedrock, Azure,
  Mistral, DeepSeek, Together, Fireworks, Groq, xAI, Cerebras,
  SambaNova, OpenRouter, Ollama, vLLM, llama.cpp, any
  OpenAI-compatible URL — one `[models.X]` block per provider,
  one virtual function name per call site.
- **Self-hostable end to end.** Gateway + ClickHouse + UI all
  run on your hardware; egress is your model providers only;
  no SaaS dependency for any feature in OSS.
