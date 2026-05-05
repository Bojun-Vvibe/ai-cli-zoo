# drill

> **Single-binary HTTP load-testing CLI driven by a YAML
> benchmark file** — describe a sequence of HTTP requests
> (with assertions, captured variables, dependent steps,
> ramp-up / ramp-down profiles, and per-iteration data from
> a CSV) in one declarative document, then `drill --benchmark
> spec.yml` runs N concurrent users for M iterations and
> prints per-step latency histograms plus pass/fail counts.
> Pinned to **v0.9.0** (released 2024-09-15,
> [LICENSE](https://github.com/fcsonline/drill/blob/0.9.0/LICENSE),
> GPL-3.0).

Source: <https://github.com/fcsonline/drill>

## TL;DR

The HTTP load-test CLI space splits into (a) imperative
scripting tools where you write JavaScript / Lua / Go to
describe the user journey — [`k6`](../k6/) (JS),
`wrk` + Lua, `gatling` (Scala), `locust` (Python); and (b)
declarative tools where the journey is a flat config file
and the runner does the orchestration —
[`vegeta`](../vegeta/) (one-line targets file, no flow),
`oha` / `bombardier` (single endpoint, no flow), Apache
`ab` (single endpoint). `drill` sits in the gap between
them: it is *declarative* (YAML, no programming language
to learn) but *flow-aware* (each step can capture a value
from the previous response and feed it as a path / header /
body parameter into the next), which means realistic
multi-step user journeys — login, fetch token, list items,
post item, delete item — fit in 30 lines of YAML and run
at thousands of concurrent users from a single binary, no
runtime, no Node, no JVM.

## Install

```bash
# macOS (Homebrew)
brew install drill

# Cargo (any platform with Rust 1.70+)
cargo install drill

# Arch Linux (AUR)
yay -S drill

# Prebuilt binaries: GitHub releases
curl -L https://github.com/fcsonline/drill/releases/download/0.9.0/drill-0.9.0-x86_64-unknown-linux-gnu.tar.gz \
  | tar -xz && sudo mv drill /usr/local/bin/

# Verify
drill --version           # drill 0.9.0
```

## License

GPL-3.0 — see
[LICENSE](https://github.com/fcsonline/drill/blob/0.9.0/LICENSE).
Copyleft: redistributing modified `drill` (or a tool that
embeds the `drill` library) requires the same licence; the
*YAML benchmarks* you write and the *load-test reports* you
generate are your own and not encumbered. The runtime is a
single Rust-compiled binary — no Python / Node / JVM
dependency, drops cleanly into a CI runner image.

## Common invocations

```bash
# Run a benchmark file once (single user, single iteration)
drill --benchmark bench.yml

# 100 concurrent users, 50 iterations each
drill --benchmark bench.yml --concurrency 100 --iterations 50

# Stats only (no per-request line), exit non-zero on failed asserts
drill --benchmark bench.yml --stats --quiet

# Output JSON report for CI parsing
drill --benchmark bench.yml --report report.json

# Stop on first failed assertion (good for smoke tests)
drill --benchmark bench.yml --no-keep-alive --stats
```

A representative `bench.yml`:

```yaml
---
concurrency: 50
base: 'https://api.example.org'
iterations: 200
rampup: 10        # seconds to reach full concurrency

plan:
  - name: Login and capture token
    request:
      url: /auth/login
      method: POST
      body: '{"user":"bench","pass":"bench"}'
      headers:
        Content-Type: application/json
    assign: login

  - name: List items (auth required)
    request:
      url: /items
      headers:
        Authorization: 'Bearer {{ login.body.token }}'
    assign: items

  - name: Fetch each item by id
    request:
      url: '/items/{{ item }}'
      headers:
        Authorization: 'Bearer {{ login.body.token }}'
    with_items_from_csv: ./ids.csv
```

## Pipeline patterns this enables

- **Realistic journey load tests in CI**: commit the YAML
  next to the API spec, run `drill --benchmark
  bench.yml --stats --quiet` as a release-gate job — fail
  the build when p99 of the login step crosses a
  threshold, without writing a line of JS or Lua.
- **Reproducible regressions across versions**: bisect
  performance with `git bisect run drill --benchmark
  bench.yml --stats` because the YAML is version-controlled
  alongside the code being tested.
- **Composes with the rest of the zoo**: chase a regression
  flagged by `drill` with [`hyperfine`](../hyperfine/) for
  the single-endpoint micro-benchmark and
  [`bandwhich`](../bandwhich/) /
  [`trippy`](../trippy/) to attribute network share — and
  pipe `drill --report report.json` into [`jq`](../jaq/) +
  [`youplot`](../youplot/) for an at-a-glance latency
  histogram in the build log.

## When NOT to use it

- You need **complex stateful logic** in the user journey —
  branching on response codes, cryptographic signing per
  request, custom protocol framing — write a
  [`k6`](../k6/) script in JS or use Locust in Python; YAML
  declarativity has a ceiling.
- You need to test **non-HTTP protocols** (gRPC, WebSocket
  streams, MQTT, raw TCP) — `drill` is HTTP-only by design;
  reach for `ghz` (gRPC), `artillery` (multi-protocol), or
  `tsung`.
- You need **distributed load generation** across many
  worker nodes for million-rps tests —
  [`k6`](../k6/) Cloud, `gatling` Enterprise, `locust`
  master/worker mode are the right tools; `drill` runs on
  one machine.

## Why it earns a slot in the zoo

The zoo already has the single-endpoint hammers
([`vegeta`](../vegeta/), [`k6`](../k6/)) and the heavy
scripting frameworks. `drill` fills the niche between them:
**declarative, flow-aware, dependency-tracked HTTP load
tests in a single Rust binary**, with a config format you
can read at a glance and a learning curve measured in
minutes. It is the right answer for "I want a CI-friendly
load test of a 4-step journey, I do not want to learn JS,
and I do not want to install a runtime."
