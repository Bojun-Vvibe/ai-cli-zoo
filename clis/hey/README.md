# hey

- **Repo:** https://github.com/rakyll/hey
- **Version:** v0.1.5 (latest tag, the canonical stable release)
- **License:** Apache-2.0 ([LICENSE](https://github.com/rakyll/hey/blob/master/LICENSE))
- **Language:** Go
- **Install:** `go install github.com/rakyll/hey@latest` · `brew install hey` ·
  prebuilt binaries on the GitHub release page

## What it does

`hey` is a tiny HTTP load generator written in Go — the spiritual
successor to Apache's `ab` (`ApacheBench`) by Jaana Dogan, with
TLS / HTTP/2 / keep-alive / cookies / arbitrary headers and bodies done
right and a single static binary as the install. The mental model is
deliberately simple: `hey -n <total-requests> -c <concurrent-workers>
<url>` fires `n` requests with `c` workers in flight, then prints a
classic latency-percentile breakdown (p10 / p25 / p50 / p75 / p90 / p95 /
p99), a histogram, a status-code summary, and per-stage timing (DNS,
TCP connect, TLS, server processing). `-z <duration>` (e.g. `-z 30s`)
swaps the trip count for a wall-clock cap so you can say "hammer this
URL for thirty seconds at 200 connections and tell me what broke." A
small handful of flags cover almost every real shape:

- `-m <method>` and `-d <body>` / `-D <body-file>` for non-GET workloads.
- `-H "Authorization: Bearer …"` (repeatable) for auth headers.
- `-T 'application/json'` for content-type.
- `-q <qps-per-worker>` to *cap* throughput rather than saturate it.
- `-disable-keepalive` / `-disable-redirects` / `-disable-compression`
  for testing pathological clients.
- `-cpus N` to bound the load generator's own CPU footprint.
- `-o csv` for machine-readable output you can pipe into `gnuplot`,
  `pandas`, or a CI gate.

It is the **closed-model** loading shape: workers keep one request in
flight each and start the next as soon as the previous returns, so under
load the achieved RPS is whatever the *server* can sustain rather than
an arrival-rate target. This is the right shape for "how fast can my
service actually go" measurement; it is the wrong shape for SLO-style
"what does p99 look like at 500 RPS *whether or not the server is
healthy*" measurement (use [`vegeta`](../vegeta/) for that — vegeta is
constant-rate / open-model and survives coordinated omission).

## When to pick it / when not to

Pick `hey` when you want a `wrk`-class one-liner that does not need a
Lua script, runs anywhere Go runs (Windows / macOS / Linux / *BSD,
amd64 / arm64), and outputs numbers any reviewer can read. The default
`hey -n 200 -c 50 https://api.example.com/v1/health` is the universal
"is this endpoint alive under realistic concurrency" smoke test, and
the histogram + percentile output is enough to back a "the change made
p95 go from 80 ms to 320 ms" PR comment. It is also the right shape for
quick saturation experiments — pick the worker count, watch the
achieved RPS plateau, that plateau is your service's ceiling for that
request type. The static-Go-binary install means `hey` works on a
locked-down jump box where you cannot install Python (`locust`), Java
(`jmeter`), or Lua (`wrk`).

Skip it for **scenario-based load tests** where one virtual user logs
in, browses three pages, posts a comment, then logs out — that is
[`k6`](../k6/)'s shape (full JS scenario DSL with stages and
checks). Skip it for **SLO measurement at a fixed arrival rate** —
`hey` is closed-model, [`vegeta`](../vegeta/) is open-model with
`-rate=200/s` semantics that do not deform when the server slows
down. Skip it for **HTTP/3 (QUIC) targets** — `hey` speaks HTTP/1.1
and HTTP/2 only; [`oha`](../oha/) (HTTP/1/2/3 with a real-time TUI) or
`bombardier` are better picks for QUIC. Skip it for **WebSocket / gRPC /
TCP / MQTT** — wrong protocol; reach for [`grpcurl`](../grpcurl/) +
`ghz` for gRPC, [`websocat`](../websocat/) for WebSocket, [`mqttx`](../mqttx/)
for MQTT, [`plow`](../plow/) for an HTTP/1+2 closed-model load tool with
a live TUI. And skip it for **distributed load** — `hey` is single-host;
once you saturate one box's network or epoll, you need `locust` /
`k6 cloud` / `gatling` / a fleet of `vegeta` workers behind a coordinator.

The repo has been quiet for several years (v0.1.5 has been the canonical
release for a long time), but `hey` is feature-complete for what it
does — there is no roadmap because there is no missing feature in
"give me HTTP load with percentile output." Treat it as a stable
classic, not a living project.

## AI-native angle

`hey` is one of the cleanest "agent-driven perf gate" primitives in
the catalog because the output is shaped for parsing:

- **`-o csv` is the integration surface.** `hey -n 1000 -c 50 -o csv
  https://… > before.csv` and `… > after.csv` are two artifacts an
  agent can diff to make a "p99 regressed by 12%" claim with a
  reproducible script.
- **CI gate one-liner.** `hey -n 500 -c 25 https://staging/$endpoint
  -o csv | awk -F, 'NR>1 {sum+=$1; n++} END {print sum/n}'` gives the
  mean response time as one number an agent can compare against a
  baseline; pair with [`hyperfine`](../hyperfine/) for cold-start
  latency of the request issuer itself.
- **Pairs with [`samply`](../samply/) for attribution.** `hey` tells
  you the *server* slowed down; `samply record -p $(pgrep myserver)`
  during the next `hey` run tells you *which function* slowed down.
- **No telemetry, no account, no daemon.** `hey` makes no outbound
  call other than to the URL you gave it; safe to run from any agent
  context including air-gapped boxes.

## Alternatives

- **[`vegeta`](../vegeta/)** — constant-rate / open-model HTTP load
  testing. Pick `vegeta` for SLO testing where coordinated omission
  matters; pick `hey` for closed-model saturation tests where "max
  RPS this server can do" is the question.
- **[`oha`](../oha/)** — Rust port of `hey`'s shape with a real-time
  TUI showing latency histogram and per-second RPS while the run is
  in flight, plus HTTP/3 (QUIC) support. Same closed-model shape.
  Pick `oha` if you want to *watch* the run and need HTTP/3; pick
  `hey` for the smaller binary and the boring-stable Go ecosystem.
- **[`bombardier`](../bombardier/)** — single-binary Go HTTP load
  tester with a live progress bar and slightly better defaults
  around connection pooling. Same shape as `hey` for the most
  common cases; pick whichever is already on the box.
- **[`plow`](../plow/)** — closed-model Go load tool with a live TUI
  and HTTP/1+2 support. The "I want oha's UX but a stable Go
  toolchain" middle ground.
- **[`wrk`](https://github.com/wg/wrk)** — the C-language classic;
  faster generator (saturates a 10 GbE NIC easily) but the Lua
  scripting hook adds friction the moment you want a non-trivial
  request shape. Pick `wrk` when raw load-generator throughput
  matters; pick `hey` when you want the same job done in one
  command line.
- **[`k6`](../k6/)** — JS-scenario load testing with stages,
  thresholds, output backends, and a hosted dashboard option.
  Different layer: `k6` is for full user-journey load tests, `hey`
  is for hammering one URL.
- **[`ali`](../ali/)** — terminal load-test tool with a live latency
  plot rendered in the TUI. `hey` minus the TUI plus standard
  percentile output.
- **[`apache-bench`](https://httpd.apache.org/docs/current/programs/ab.html)
  / `ab`** — the original. `hey` exists because `ab`'s output is
  hostile, its TLS support is broken, and HTTP/2 is missing. Skip
  `ab` unless it is the only thing on the box.
