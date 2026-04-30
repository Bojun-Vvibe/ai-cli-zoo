# fluent-bit

- **Repo:** https://github.com/fluent/fluent-bit
- **Version:** v5.0.3
- **License:** [LICENSE](https://github.com/fluent/fluent-bit/blob/master/LICENSE) (Apache-2.0)
- **Category:** Observability pipeline / lightweight log+metric+trace forwarder

## What it is

Fluent Bit is a CNCF-graduated telemetry agent written in C, optimized for
sub-megabyte memory and microsecond-class throughput on edge nodes,
Kubernetes DaemonSets, and embedded devices. It speaks the same
`inputs` → `filters` → `outputs` pipeline model as its older sibling
Fluentd but with a much smaller surface: ~100 native plugins for sources
(tail, systemd, kubelet, OTLP, HTTP, Kafka, OpenTelemetry), filters
(parser, modifier, Lua, Wasm, multiline), and sinks (Loki, Elasticsearch,
S3, OpenSearch, Datadog, OTLP, Splunk, ClickHouse). Configuration is
either the classic `.conf` INI dialect or the newer YAML form.

## Why it fits an AI-native workflow

- **Cheapest possible sidecar for inference pods** — a 2 MB RSS binary next
  to a 14 GB model server collects stdout/stderr, parses JSON-line traces
  from your agent framework, attaches Kubernetes metadata, and ships to
  any backend without stealing the GPU node's RAM budget.
- **Wasm and Lua filters for in-flight redaction** — run a small Wasm
  module that scrubs PII or trims overlong prompt fields *inside* the
  agent process, so the egress payload (and your vendor bill) stays bounded
  even when an LLM service goes into a retry storm.
- **First-class OpenTelemetry input and output** — sits cleanly in front of
  or behind an OpenTelemetry Collector, which lets you keep Fluent Bit at
  the edge for tiny footprint and use the heavier Collector only at
  regional aggregation tiers where you actually need its processor zoo.
