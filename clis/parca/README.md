# parca

- **Repository:** https://github.com/parca-dev/parca
- **Latest version:** v0.27.1
- **License:** Apache-2.0 — verified at [`LICENSE`](https://github.com/parca-dev/parca/blob/main/LICENSE) (raw: https://raw.githubusercontent.com/parca-dev/parca/main/LICENSE)
- **Niche:** Always-on, low-overhead **continuous profiling** of running
  processes — CPU, allocations, off-CPU — across an entire fleet, with
  the same query/diff UX you expect from a metrics stack.

## What it does

`parca` runs a server (`parca`) that scrapes pprof-shaped profile
endpoints (or receives them from the eBPF agent
[`parca-agent`](https://github.com/parca-dev/parca-agent)), stores them
in a columnar profile database (FrostDB), and serves a UI + PromQL-like
query API over them. The CLI surface is small but operational:

```
# Run the server locally against a config file that lists scrape targets
parca --config-path=parca.yaml --storage-active-memory=1073741824

# Point an agent at a node to harvest CPU profiles via eBPF (no app changes)
parca-agent --node=laptop --remote-store-address=localhost:7070 --remote-store-insecure

# In the UI: pick a profile type (process_cpu:cpu:nanoseconds:cpu:nanoseconds),
# filter by labels (job="api", pod=~"api-.*"), and diff two time ranges to
# see exactly which stack frames got more expensive between deploys.
```

## Why interesting

Almost every team has metrics (request rate, p99) and logs (the error
text), and almost no team has *profiles* — the actual stack traces of
where CPU time and allocations are going right now in production.
Historically that gap existed because profiling meant either (a)
SSH-ing in and running `perf` for 30 seconds during an incident, or
(b) instrumenting the binary with `net/http/pprof` and remembering to
hit `/debug/pprof/profile` while the bug was happening. Both are
reactive and both lose the long tail.

`parca` closes the gap by treating profiles like metrics: scraped on
an interval, kept in a queryable store, joinable on labels (`job`,
`pod`, `version`), and *diffable across time*. The killer query shape
is "show me which stack frames got 30% slower between v142 and v143"
— answerable in the UI in under a minute, where previously it was a
multi-day bisect.

The eBPF agent is the second half of the trick: it harvests CPU
profiles from *every* process on a node — Go, C, C++, Rust, Java
(with [JFR](https://github.com/parca-dev/parca-agent) frame
resolution) — without recompiling, without a sidecar, and at sub-1%
overhead. So you get a profile of your app, your sidecars, your
runtime, your kernel time, and your noisy neighbours in the same
flame graph, all addressable by the labels you already use in
Prometheus / Grafana.

## Pairs well with

- **Prometheus / Grafana / [`vector`](../vector/) /
  [`otel-collector`](../otel-collector/)** — `parca` is the missing
  third leg (profiles) of the metrics + logs + traces stool. Same
  label model, same scrape semantics, same incident workflow.
- **[`pprof`](https://github.com/google/pprof)** — `parca` speaks
  pprof natively; you can `go tool pprof http://parca/api/v1/...` a
  stored profile and use the Go toolchain's flamegraph / source view
  on it.
- **[`hyperfine`](../hyperfine/)** for *micro*-benchmarks of a single
  command vs. `parca` for *macro* observation of a long-running fleet
  — opposite ends of the same "where is the time going" question.

## When to skip

- Single-binary, single-laptop dev work — `go test -cpuprofile` plus
  `go tool pprof` is faster than standing up a `parca` server and an
  agent for a 5-second flame graph.
- You only ever profile during incidents and you already SSH into the
  box — `perf record` + `flamegraph.pl` is one command and zero
  infrastructure; `parca`'s value is the *continuous* part.
- You need *tracing* (per-request span timelines), not profiling
  (aggregate stack-frame time) — those are different questions; reach
  for Tempo / Jaeger / the [`otel-collector`](../otel-collector/)
  pipeline instead.
