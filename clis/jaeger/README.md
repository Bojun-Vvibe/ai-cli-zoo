# jaeger

> **The reference distributed-tracing platform under the CNCF
> umbrella — a single Go binary (`jaeger`) that ingests OTLP
> spans over gRPC :4317 and HTTP :4318, stores them in
> in-memory / Badger / Cassandra / Elasticsearch / OpenSearch /
> ClickHouse, and serves a query API + web UI on :16686 so a
> request that hops across 30 services lands as one
> waterfall.** Pinned to **v2.17.0** (released 2026-03-30),
> [LICENSE](https://github.com/jaegertracing/jaeger/blob/main/LICENSE),
> Apache-2.0.

Source: <https://github.com/jaegertracing/jaeger>

## TL;DR

`jaeger` (v2 line) is now a single all-in-one binary built on
top of the OpenTelemetry Collector — the legacy
`jaeger-agent` / `jaeger-collector` / `jaeger-query` /
`jaeger-ingester` split is gone, replaced by one process whose
behaviour is driven entirely by a YAML config (`jaeger
--config config.yaml`). Out of the box it speaks **OTLP**
(the OpenTelemetry wire format) on gRPC :4317 and HTTP :4318,
plus the legacy Jaeger Thrift / gRPC ports for backwards
compatibility with apps still using the old
`jaeger-client-*` SDKs. Spans go through the OTel Collector
pipeline (receivers → processors → exporters) so you can
batch, filter, sample, redact, and route to multiple backends
in the same process. The query side serves a JSON API on
:16686 (`GET /api/services`, `GET /api/traces?service=...`)
plus the React UI in the same binary, so a `docker run -p
16686:16686 -p 4317:4317 jaegertracing/jaeger:2.17.0` is a
fully working tracing stack with no extra moving parts —
ideal for local dev, CI assertions on trace shape, and small
production deployments. For larger setups, swap the in-memory
store for Cassandra / Elasticsearch / OpenSearch /
ClickHouse via config; the same `jaeger` binary becomes
collector-only or query-only depending on which pipelines
you enable.

## Install

```bash
# Static binary from GitHub Releases (linux / macOS / windows, amd64 / arm64)
curl -fsSL -o /tmp/jaeger.tar.gz \
  https://github.com/jaegertracing/jaeger/releases/download/v2.17.0/jaeger-2.17.0-darwin-arm64.tar.gz
tar -xzf /tmp/jaeger.tar.gz -C /tmp
sudo install /tmp/jaeger-2.17.0-darwin-arm64/jaeger /usr/local/bin/

# Docker (the supported all-in-one path)
docker run --rm --name jaeger \
  -p 16686:16686 \
  -p 4317:4317 -p 4318:4318 \
  -p 5778:5778 -p 9411:9411 \
  jaegertracing/jaeger:2.17.0

# Homebrew (community-maintained tap)
brew install jaeger

# Verify
jaeger --version    # jaeger version v2.17.0

# First-time run with default in-memory storage
jaeger --config <(cat <<'EOF'
service:
  extensions: [jaeger_storage, jaeger_query, healthcheckv2]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger_storage_exporter]
EOF
)
# UI now on http://127.0.0.1:16686
```

## One Concrete Example

```bash
# Spin up jaeger, send 10 traces from a Python app via the
# OpenTelemetry SDK, find the slow ones from the CLI.

# 1. Run jaeger all-in-one in the background
docker run -d --rm --name jaeger \
  -p 16686:16686 -p 4317:4317 \
  jaegertracing/jaeger:2.17.0

# 2. Point any OTel SDK at :4317 — Python example
pip install opentelemetry-api opentelemetry-sdk \
            opentelemetry-exporter-otlp-proto-grpc

cat <<'PY' > app.py
from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
import time, random

tp = TracerProvider(resource=Resource.create({"service.name": "checkout"}))
tp.add_span_processor(BatchSpanProcessor(
    OTLPSpanExporter(endpoint="http://127.0.0.1:4317", insecure=True)))
trace.set_tracer_provider(tp)
tracer = trace.get_tracer(__name__)

for i in range(10):
    with tracer.start_as_current_span("place-order") as span:
        span.set_attribute("order.id", i)
        with tracer.start_as_current_span("charge-card"):
            time.sleep(random.uniform(0.05, 0.4))
        with tracer.start_as_current_span("write-db"):
            time.sleep(random.uniform(0.01, 0.2))
tp.shutdown()
PY
python app.py

# 3. List services + slowest traces from the query API
curl -s http://127.0.0.1:16686/api/services | jq '.data'
# ["checkout"]

curl -s "http://127.0.0.1:16686/api/traces?service=checkout&limit=20" \
  | jq '.data[] | {trace: .traceID, dur_ms: (.spans[0].duration/1000)}' \
  | jq -s 'sort_by(-.dur_ms) | .[:3]'

# 4. Open the UI to see the waterfall
open http://127.0.0.1:16686
```

For production, swap the storage:

```yaml
# config.yaml — Elasticsearch backend
extensions:
  jaeger_storage:
    backends:
      some_storage:
        elasticsearch:
          server_urls: [http://es:9200]
          indices:
            spans: { name: jaeger-span, date_layout: "2006-01-02" }
```

## License

[Apache-2.0](https://github.com/jaegertracing/jaeger/blob/main/LICENSE),
SPDX `Apache-2.0`. CNCF graduated project.

## Niche / positioning

The **CNCF-graduated OSS distributed-tracing backend** — pick
`jaeger` over hosted SaaS tracers (Datadog APM / Honeycomb /
New Relic / Lightstep) when the trace store needs to live
inside your perimeter (regulated data, on-prem, air-gapped)
and over [`vector`](../vector/) when you want a tracing
backend with a query UI and not just a routing pipeline.
Pick over [`otel-cli`](../otel-cli/) when you need a *backend*
to receive and visualise spans (otel-cli is a span *emitter*
for shell scripts — they compose: `otel-cli` → OTLP →
`jaeger`). Pick over Zipkin (also OSS) when you want OTLP-
native ingest, the OTel Collector pipeline as the data plane,
and a more actively maintained UI; v2's "everything is the
OTel Collector" architecture means any OTel processor /
exporter (sampling, redaction, S3 export, Kafka mirror) is a
config block, not a custom binary. Skip when your scale needs
metrics + logs + traces in one product (use Grafana LGTM /
SigNoz / Datadog), when you need long-term trace retention
without managing Cassandra / Elasticsearch yourself (use a
hosted backend), or when the team has already standardised
on Tempo (Grafana's tracing backend, also OTLP-native, deeper
Grafana integration).
