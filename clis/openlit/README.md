# openlit

- **Repo:** https://github.com/openlit/openlit
- **Version:** openlit-1.18.1 (latest SDK release; HEAD pinned 295d680)
- **License:** Apache-2.0 (`LICENSE`)
- **Category:** LLM observability / OpenTelemetry instrumentation

## What it does

An OpenTelemetry-native observability platform for LLM apps, agents, and
GPU workloads. Ships a one-line auto-instrumentation SDK
(`openlit.init()`) that captures spans, token counts, costs, and prompts
across 50+ providers (OpenAI, Anthropic, Bedrock, Ollama, vLLM, etc.),
vector DBs, and agent frameworks; pairs with a self-hostable UI for
traces, evals, prompt management, and a secrets vault.

## Install

```sh
pip install openlit
# or run the full stack
git clone https://github.com/openlit/openlit && cd openlit/docker-compose
docker compose up -d
```

## Why it's interesting

Most LLM-tracing tools ship a proprietary wire format and lock you into
their backend. openlit emits standard OTel spans with semantic conventions
proposed for GenAI, so the same instrumentation feeds Grafana, Jaeger,
Datadog, or the bundled UI without code changes. The combination of
auto-instrumentation, GPU monitoring, and an OSS playground/eval surface
in one repo makes it a credible self-hosted alternative to Helicone,
Langfuse, or Phoenix when you want to keep prompts on your own infra.
