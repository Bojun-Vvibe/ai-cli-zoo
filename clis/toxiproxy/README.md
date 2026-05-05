# toxiproxy

> **Programmable TCP proxy for chaos / resiliency testing: inject
> latency, bandwidth caps, slow closes, and connection resets
> between your service and its dependencies, scripted from the
> CLI or HTTP API.** Pinned to **v2.12.0**
> ([LICENSE](https://github.com/Shopify/toxiproxy/blob/main/LICENSE),
> MIT).

Source: <https://github.com/Shopify/toxiproxy>

## TL;DR

`toxiproxy` is a two-binary system: `toxiproxy-server` (a long-lived
TCP proxy daemon with an HTTP control plane) and `toxiproxy-cli`
(the operator surface for creating proxies and attaching
"toxics"). You point your application at the proxy port instead
of the real upstream (Postgres, Redis, an internal HTTP API),
then use the CLI to attach toxics like `latency`, `bandwidth`,
`slow_close`, `timeout`, `slicer`, or `limit_data`. The proxy
modifies the byte stream in-flight, so your application sees
exactly the network conditions you specified — no kernel
`tc`/`netem` rules, no privileged container, no SRE ticket. Built
and used in production at Shopify for failure-injection tests in
CI; the design point is "deterministic, scriptable, in-test
dependency degradation," not general traffic shaping.

## Install

```bash
# Homebrew (server + CLI)
brew install toxiproxy

# Go install
go install github.com/Shopify/toxiproxy/v2/cmd/toxiproxy-server@v2.12.0
go install github.com/Shopify/toxiproxy/v2/cmd/toxiproxy-cli@v2.12.0

# Docker (server only)
docker run --rm -p 8474:8474 -p 26379:26379 ghcr.io/shopify/toxiproxy:2.12.0

# verify
toxiproxy-cli --version
```

## Examples

```bash
# Start the control plane (separate terminal or background process)
toxiproxy-server &

# Create a proxy in front of Redis: app connects to :26379, proxy talks to real Redis on :6379
toxiproxy-cli create -l 127.0.0.1:26379 -u 127.0.0.1:6379 redis

# Inject 1s of latency on every byte going downstream (Redis -> app)
toxiproxy-cli toxic add -t latency -a latency=1000 -n slow_redis redis

# Simulate a flaky link: 10% of bytes get a 500ms hold
toxiproxy-cli toxic add -t latency -a latency=500 --toxicity 0.1 redis

# List active toxics, then remove one
toxiproxy-cli inspect redis
toxiproxy-cli toxic remove -n slow_redis redis
```

## When to choose it

Pick `toxiproxy` when you need **deterministic, in-test**
network degradation between an application and a single
upstream dependency, and you want the configuration to live in
the test suite (Ruby/Go/Python/Node clients exist) rather than
in `iptables` rules. Canonical fits: integration tests that
must verify retry / circuit-breaker behavior under latency,
chaos experiments inside a CI job, reproducing a customer's
"sometimes Postgres takes 4s" bug locally.

Skip it when you need cluster-wide chaos (`chaos-mesh`,
`litmus`, `gremlin` are the right shape — they manipulate
kernel netfilter / pod networking via DaemonSets), when the
failure mode is at L7 (return malformed JSON, 500 errors —
write a stub server instead, the proxy operates on raw bytes),
or when you need to shape *all* traffic from a host (use `tc
qdisc netem` directly).

## Vs adjacent tools

- **Vs `tc qdisc netem`:** `netem` lives in the kernel, applies
  to an interface, and requires `CAP_NET_ADMIN`. Powerful but
  global, hard to scope per-test, and configuration is not
  portable across Linux distros. `toxiproxy` is per-flow,
  user-space, and the same config runs on macOS dev laptops
  and Linux CI runners.
- **Vs `chaos-mesh` / `litmus`:** those are Kubernetes-native
  and operate on pods / services / namespaces. The right shape
  for "kill a random pod every 30 minutes." `toxiproxy` is the
  right shape for "this single test must observe a 200ms
  latency spike at byte 4096 of the response."
- **Vs application-layer mocks (`wiremock`, `mockoon`):** mocks
  *replace* the upstream. `toxiproxy` keeps the real upstream
  and only degrades the wire. Use mocks when you want to
  control *what* the upstream returns; use `toxiproxy` when
  you want to control *how* the bytes arrive.

## Caveats

- **TCP only.** UDP, QUIC, and HTTP/3 are out of scope. If
  your dependency is gRPC-over-HTTP/2 (TCP) you are fine; if
  it is QUIC, look elsewhere.
- **The CLI is operator-facing.** In test code, you almost
  always want the language-native client
  (`github.com/Shopify/toxiproxy/v2/client` for Go,
  `toxiproxy-ruby` gem, etc.) — it talks the same HTTP API as
  the CLI, but with proper teardown semantics in `defer` /
  `after` blocks. The CLI is for ad-hoc exploration.
- **One server, many proxies.** A single `toxiproxy-server`
  process holds all proxy state in memory. If two test suites
  share the same server, they will fight over proxy names.
  In CI, run a server per job, not a global one.
- **Toxic ordering matters.** Toxics are applied in the order
  attached. `latency` then `bandwidth` ≠ `bandwidth` then
  `latency` for bursty payloads. The `inspect` output shows
  the chain — read it before debugging "why didn't my toxic
  fire."
