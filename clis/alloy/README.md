# alloy

> **OpenTelemetry-Collector-compatible telemetry agent with a
> programmable pipeline language** — a single Go binary from Grafana
> Labs that ingests metrics, logs, traces, and profiles, transforms
> them through a declarative `.alloy` config (the River-derived
> configuration language with components, expressions, and live
> reloading), and ships them to any OTLP / Prometheus-remote-write /
> Loki / Tempo / Pyroscope / Mimir endpoint. Pinned to **v1.16.0**
> (release published 2026-04-23,
> [LICENSE](https://github.com/grafana/alloy/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/grafana/alloy>.

## TL;DR

`alloy` is what you reach for when a vanilla `otelcol` YAML pipeline
has stopped scaling — when you need conditional routing
("send debug-level logs to one backend, error-level to another"),
secret interpolation from a remote KMS, in-process discovery
(Kubernetes pod-watching, EC2 tag matching, Consul service lists),
hot-reloadable config without restarting the agent, and a single
binary that simultaneously scrapes Prometheus exporters, tails
container stdout, receives OTLP, runs eBPF-based auto-instrumentation,
and forwards profiles. Internally it is the OpenTelemetry Collector
plus the Prometheus Agent plus the Loki Promtail tailer plus the
Pyroscope eBPF profiler, fused into one process and wired together
through River — so a single `alloy run config.alloy` replaces the
"three sidecars per pod" pattern with one DaemonSet.

## Install

```bash
# Homebrew
brew install grafana/grafana/alloy

# Linux / macOS tarball
curl -L https://github.com/grafana/alloy/releases/download/v1.16.0/alloy-darwin-arm64.zip \
  -o alloy.zip && unzip alloy.zip && sudo install alloy-darwin-arm64 /usr/local/bin/alloy

# Container
docker pull grafana/alloy:v1.16.0

# verify
alloy --version    # v1.16.0
```

Official Helm chart (`grafana/alloy`) and Kubernetes operator
(`alloy-operator`) cover the cluster install path; the binary
distribution is enough for a single-node pipeline or local
development against a `compose.yaml` of backends.

## License

Apache-2.0 — see [LICENSE](https://github.com/grafana/alloy/blob/main/LICENSE).
Permissive; redistributable in commercial pipelines, attribution
in the binary's `--help`.

## One Concrete Example

```alloy
// config.alloy — scrape node-exporter, tail container logs, ship
// metrics to Prometheus-remote-write and logs to Loki.

prometheus.scrape "node" {
  targets    = [{ __address__ = "localhost:9100" }]
  forward_to = [prometheus.relabel.add_env.receiver]
  scrape_interval = "15s"
}

prometheus.relabel "add_env" {
  forward_to = [prometheus.remote_write.primary.receiver]
  rule {
    target_label = "env"
    replacement  = sys.env("DEPLOY_ENV")
  }
}

prometheus.remote_write "primary" {
  endpoint {
    url = "https://prom.example.com/api/v1/write"
    basic_auth {
      username = "scraper"
      password = sys.env("PROM_RW_TOKEN")
    }
  }
}

loki.source.docker "containers" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.docker.containers.targets
  forward_to = [loki.write.primary.receiver]
}

discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

loki.write "primary" {
  endpoint {
    url = "https://loki.example.com/loki/api/v1/push"
  }
}
```

```bash
# run with hot reload
alloy run --server.http.listen-addr=0.0.0.0:12345 config.alloy

# edit config.alloy, then:
curl -X POST http://localhost:12345/-/reload

# inspect the live component graph
open http://localhost:12345/graph
```

## Niche It Fills

**One agent, many telemetry types, one config language.** The
classic stack is `prometheus` (or `vmagent`) for metrics +
`promtail` (or `fluent-bit`) for logs + `otelcol` for traces +
optionally a profiler — each with its own YAML, its own scrape
syntax, its own service-discovery quirks, its own Helm chart.
`alloy` collapses that into a single binary whose pipeline graph
is expressed in River, a Terraform-flavored DSL with components,
typed expressions, secret references, and live reload. The
components are upstream `otelcol` factories under the hood (so
every OTel processor / exporter is reachable), but with first-class
metric scrape and log tail components grafted on. For ops teams
running observability pipelines on Kubernetes that already speak
the Grafana stack, this drops process count, memory footprint, and
config-file count in one step.

## Why use it

Three concrete things that pay back the migration from `otelcol +
promtail + vmagent`:

1. **One DaemonSet instead of three.** A typical Kubernetes
   observability sidecar set is `node-exporter` + `promtail` +
   `otel-collector`; `alloy` replaces the latter two and pairs with
   `node-exporter` directly. Fewer pods, fewer Helm releases,
   fewer config-map churns on rollout.
2. **River > YAML for non-trivial pipelines.** Conditional routing,
   secret references, `for` loops over Kubernetes namespaces, and
   inline expressions (`replacement = "{{ .pod }}-{{ env \"REGION\" }}"`)
   are all first-class language features instead of YAML anchors
   and `tpl` Helm hacks. The component graph is introspectable at
   `/graph` and reloadable without restart.
3. **Profiles + traces + logs + metrics in one wire format.**
   `alloy` ships a Pyroscope-compatible profiler (`pyroscope.scrape`,
   `pyroscope.ebpf`) alongside OTLP receivers, so you can light up
   continuous profiling without deploying a second agent. For
   teams adopting the Grafana LGTM stack (Loki / Grafana / Tempo /
   Mimir) plus Pyroscope, a single agent terminates all four
   pipelines.

For an LLM-driven observability use case ("agent investigates a
trace, follows it to the logs, then to the metrics"), having one
collector that emits all three signals with consistent resource
attributes makes the cross-signal join trivial — every span, log,
and metric carries the same `service.name` / `k8s.pod.uid` because
the same component populated them.

## Vs Already Cataloged

- **Vs [`otel-collector`](../otel-collector/):** closest peer.
  `otelcol` is the upstream OpenTelemetry Collector with YAML
  config and a tightly scoped charter (OTel signals only).
  `alloy` is a Grafana-Labs distribution that *embeds*
  `otelcol`'s components but adds (a) River instead of YAML,
  (b) Prometheus scraping and Loki log tailing as first-class
  components, (c) Pyroscope profiling, (d) live reload via HTTP.
  Pick `otelcol` when you want vanilla upstream and YAML-native
  ops tooling; pick `alloy` when your pipeline is non-trivial
  (conditional routing, dynamic discovery, multi-signal) or you
  are already invested in the Grafana stack.
- **Vs [`fluent-bit`](../fluent-bit/):** different center of
  gravity. `fluent-bit` is a tiny, blazing-fast log forwarder with
  metrics support bolted on; `alloy` is a multi-signal pipeline
  with logs as one component among many. For a 5-MB sidecar that
  only ships logs, `fluent-bit` wins on footprint; for a
  full-stack agent, `alloy` wins on coverage.
- **Vs [`vector`](../vector/):** closest non-OTel peer. `vector`
  is Datadog's Rust-based observability pipeline with its own VRL
  transform language and a similar "logs + metrics + traces"
  charter. The split: `vector` is Rust + VRL + unified data
  model; `alloy` is Go + River + OTLP-native. Pick `vector` for
  raw transform performance and a vendor-neutral output set; pick
  `alloy` for tight Grafana-stack integration and the OTel
  component ecosystem.
- **Vs [`otel-cli`](../otel-cli/):** different scope. `otel-cli`
  emits *one* span/event from a shell pipeline; `alloy` is the
  long-running agent that *receives* such spans and forwards
  them. They compose: `otel-cli` in CI scripts → OTLP → `alloy`
  → Tempo.

## Caveats

- **River is a learning curve.** It looks like HCL but is its own
  language with its own type system, expression syntax, and
  component model. Existing `otelcol` YAML does not translate
  one-to-one — there is a `convert` subcommand
  (`alloy convert --source-format=otelcol`) that gets you 80% of
  the way, but expect to hand-edit the rest.
- **Grafana-flavored.** The component naming, the upgrade
  cadence, the docs, and the default exporters lean toward the
  Grafana stack (Mimir / Loki / Tempo / Pyroscope). `alloy` ships
  to any OTLP backend in principle, but the path of least
  resistance is the Grafana set.
- **Resource footprint > vanilla `otelcol`.** Bundling Prometheus
  scraping, Loki tailing, and a profiler in one binary means more
  goroutines and a larger steady-state heap than a stripped-down
  `otelcol` that only handles OTLP. On constrained edge nodes
  this matters; on a typical Kubernetes worker it does not.
- **Component coverage tracks upstream `otelcol` releases on a
  lag.** A brand-new `otelcol` exporter may take a release or two
  to land in `alloy`. For exotic destinations, check the
  `alloy` component catalog before assuming parity.
- **Config language has changed name.** Older docs and blog posts
  call it "Flow mode" / "River" / "agent flow"; the project was
  renamed from "Grafana Agent" to "Grafana Alloy" in 2024 and
  Flow mode is now the only mode. Treat any "static-mode agent"
  documentation as historical.
