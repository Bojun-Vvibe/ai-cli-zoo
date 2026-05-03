# otel-desktop-viewer

> **Local OTLP receiver + browser UI for OpenTelemetry traces** —
> a single Go binary that listens on the standard OTLP gRPC/HTTP
> ports, stores received traces in memory, and serves a web UI on
> `localhost:8000` so you can inspect spans from a service you're
> developing without standing up Jaeger / Tempo / a vendor backend.
> Pinned to **v0.2.5**
> ([LICENSE](https://github.com/CtrlSpice/otel-desktop-viewer/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/CtrlSpice/otel-desktop-viewer>

## TL;DR

OpenTelemetry instrumentation works locally only if you have
*something* listening on `:4317` (gRPC) or `:4318` (HTTP) to
receive the traces — otherwise your spans go nowhere and you're
debugging blind. The standard answer is to run the OpenTelemetry
Collector + Jaeger via docker-compose, which is correct but
heavy: two containers, a config file, port plumbing, and a
backend you didn't actually want for "is my new SDK call
emitting the span I think it is?". `otel-desktop-viewer` is the
one-binary alternative: `brew install otel-desktop-viewer &&
otel-desktop-viewer` opens both OTLP ports and a web UI at
`http://localhost:8000` showing every received trace as a
flame-graph waterfall, with no persistence (memory-only,
intentionally) and no config file. When you close it, the data
is gone — that's the feature, not the bug.

## Install

```bash
# Homebrew (macOS / Linux)
brew install otel-desktop-viewer

# Go
go install github.com/CtrlSpice/otel-desktop-viewer@v0.2.5

# from a release binary
curl -L -o otel-desktop-viewer.tar.gz \
  https://github.com/CtrlSpice/otel-desktop-viewer/releases/download/v0.2.5/otel-desktop-viewer_Darwin_arm64.tar.gz
tar xf otel-desktop-viewer.tar.gz
sudo install otel-desktop-viewer /usr/local/bin/

# run — opens browser to localhost:8000, listens on :4317 (gRPC) and :4318 (HTTP)
otel-desktop-viewer

# bind to non-default ports if 4317/4318 are taken
otel-desktop-viewer --grpc-port 14317 --http-port 14318 --browser-port 18000
```

## License

Apache-2.0 — see
[LICENSE](https://github.com/CtrlSpice/otel-desktop-viewer/blob/main/LICENSE).
Permissive, patent grant included.

## One Concrete Example

```bash
# 1. start the receiver + UI
otel-desktop-viewer
# → "Listening for OTLP gRPC on :4317"
# → "Listening for OTLP HTTP on :4318"
# → "Serving UI on http://localhost:8000"
# (browser opens automatically)

# 2. point any OpenTelemetry-instrumented app at it
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
export OTEL_SERVICE_NAME=my-api
export OTEL_TRACES_EXPORTER=otlp
node ./instrumented-server.js
# every request now produces a trace visible in the UI

# 3. Python example
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
opentelemetry-instrument --traces_exporter otlp \
  --service_name my-worker \
  python worker.py

# 4. send a trace from curl (manually-crafted OTLP/HTTP)
curl -X POST http://localhost:4318/v1/traces \
  -H 'Content-Type: application/json' \
  -d @sample-trace.json
# refresh the UI — the new trace appears at the top

# 5. point an existing OTel Collector at it (tee from prod-shaped pipeline)
# in collector-config.yaml:
#   exporters:
#     otlphttp/desktop:
#       endpoint: http://localhost:4318
#   service:
#     pipelines:
#       traces:
#         exporters: [otlp/prod, otlphttp/desktop]
```

## Niche It Fills

**The "did my instrumentation actually work" feedback loop.**
Standing up Jaeger or Tempo to verify that the OTel SDK call
you just added emits the span you expected is enormous overkill;
shipping to a SaaS backend (Honeycomb, Datadog, Lightstep) leaks
dev-only data and adds latency to the iteration loop.
`otel-desktop-viewer` is the local devloop equivalent of `tail
-f` for traces: zero config, zero persistence, zero network
egress, and a flame-graph view that's good enough to confirm
"yes, the span fired, with the attributes I set, in the parent
context I expected." For an LLM-CLI workflow that adds OTel
instrumentation to user code, this is the verification
primitive — point the agent's test runs at `localhost:4318`
and inspect what came out.

## Why use it

1. **Single binary, no config, no dependencies.** Compare to
   "OTel Collector + Jaeger + docker-compose + a 60-line
   collector-config.yaml" for the same outcome at dev time.
2. **Speaks both OTLP transports on the standard ports.**
   gRPC on `:4317`, HTTP on `:4318` — the defaults of every
   OpenTelemetry SDK in every language. Setting
   `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318` is
   typically the only env var you need.
3. **Memory-only is the feature.** No disk writes, no PII
   retention, no cleanup needed — close the window and the
   trace data is gone. Safe to run against staging data, debug
   logs, customer payloads, etc., on a shared dev box.

## Vs Already Cataloged

- **Vs [`jaeger`](../jaeger/):** Jaeger is the production-grade
  trace backend (multi-tenant, persistent storage in Cassandra
  / Elasticsearch / Badger, sampling pipelines, ingest at
  scale). `otel-desktop-viewer` is the *single-developer-laptop*
  shape of the same idea: same OTLP wire protocol on the receive
  side, but in-memory, single-user, no auth, no persistence.
  Use Jaeger when traces need to outlive the process; use
  `otel-desktop-viewer` for instrumentation devloop.
- **Vs [`otel-collector`](../otel-collector/):** the Collector
  is the universal trace/metric/log *router* — it accepts OTLP
  on the receiver side and exports to N backends. It does not
  ship a UI. You can pair them: Collector receives from prod
  shape, exports a copy to `otel-desktop-viewer` for
  inspection. Use the Collector for routing; use this for
  visualisation.
- **Vs [`otel-cli`](../otel-cli/):** `otel-cli` is the
  *producer* side — emit a trace from a shell pipeline. This is
  the *receiver / viewer* side. Natural pairing: `otel-cli
  span` from a script, view the result in
  `otel-desktop-viewer`, no other infrastructure needed.
- **Vs [`jaeger`](../jaeger/) all-in-one container:** Jaeger's
  all-in-one container is the closest "single thing to run"
  alternative; it's heavier (full Jaeger UI, Badger storage,
  ~150 MB image) and persistent across restarts. Pick it when
  you want history; pick `otel-desktop-viewer` when you want
  the UI to start empty every time.

## Caveats

- **Memory-only persistence — by design.** Restart the binary
  and every trace is gone. There is no `--save` flag, no SQLite
  fallback, no replay. If you need to keep a trace, screenshot
  the flame graph or export the JSON from the UI before
  closing. (For persistent dev backends, switch to Jaeger
  all-in-one or Tempo.)
- **Single-user, no auth.** The UI on `:8000` accepts
  connections from anything that can reach the port. Don't
  expose it on `0.0.0.0` on a shared network — bind to
  `127.0.0.1` (the default) and tunnel via SSH if you need
  remote access.
- **Trace-only.** No metrics view, no log correlation. The
  receiver accepts OTLP traces; metrics and logs sent to the
  same endpoint are silently dropped. For a dev-loop log
  viewer, pair with the Collector exporting logs to a file +
  `lnav`.
- **UI is functional, not feature-complete.** Flame graph,
  span list, attribute inspector — that's it. No service map,
  no critical-path analysis, no aggregation across traces. For
  "is my span there with the right attributes" it's fine; for
  "what's the p99 latency by endpoint over the last hour" you
  want a real backend.
- **Smaller maintainer pool than the OTel Collector.** Single
  primary maintainer; release cadence is irregular (last
  release v0.2.5, August 2025). The OTLP receiver code reuses
  the upstream OTel libraries, so wire-protocol drift is
  unlikely, but UI-side feature requests can sit. The
  alternative is "switch to Jaeger all-in-one" which is a
  one-line config change.
