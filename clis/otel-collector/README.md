# otel-collector

- **Repo:** https://github.com/open-telemetry/opentelemetry-collector
- **Version:** v0.151.0
- **License:** [LICENSE](https://github.com/open-telemetry/opentelemetry-collector/blob/main/LICENSE) (Apache-2.0)
- **Category:** Observability pipeline / vendor-neutral telemetry router

## What it is

The OpenTelemetry Collector is the reference vendor-neutral agent and
gateway for the OpenTelemetry project — a single Go binary that receives
traces, metrics, and logs in any of the major wire formats (OTLP, Jaeger,
Zipkin, Prometheus, statsd, Fluent Forward, syslog), processes them
through a pipeline of `receivers` → `processors` → `exporters`, and ships
them to one or more backends. The "core" distribution ships the
foundational components; the companion `opentelemetry-collector-contrib`
repo carries the long tail of vendor-specific exporters and tail-sampling
processors. Most teams build a custom binary with `ocb`
(OpenTelemetry Collector Builder) to pin exactly the components they
use.

## Why it fits an AI-native workflow

- **Canonical landing pad for agent traces** — every modern LLM SDK
  (OpenAI, Anthropic, LangChain, LlamaIndex, semantic-kernel-style
  frameworks) emits OTLP spans for prompts, tool calls, and retries; one
  Collector deployment normalizes them and fans out to whatever trace
  backend the team already pays for.
- **Tail-sampling for token-expensive workloads** — the
  `tailsamplingprocessor` lets you keep 100% of error / high-latency /
  high-token-count agent traces while sampling cheap successes at 1%, so
  the bill scales with *interesting* traffic instead of total traffic.
- **Transform processor for prompt/PII hygiene** — OTTL (OpenTelemetry
  Transformation Language) rules redact, hash, or drop attributes
  carrying full prompts, embeddings, or customer identifiers before they
  ever reach a third-party SaaS, which makes the "can we send LLM traces
  to vendor X" review tractable.
