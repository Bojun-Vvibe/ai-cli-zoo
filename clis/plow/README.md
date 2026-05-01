# plow

## What it does
A **single-Go-binary HTTP load generator** that hammers a URL at a
fixed concurrency or a fixed RPS, computes a real latency histogram
(p50 / p75 / p90 / p95 / p99 / p99.9 + min / mean / max), groups
results by HTTP status code, and renders the live state two ways
**at the same time**: a continuously-redrawing terminal table (so
you can watch p99 climb in real time over SSH) **and** a built-in
web UI on `:18888` with sparkline charts of QPS / latency /
status-code mix that you can point a manager at without screen-
sharing your terminal. Request shape covers method, header,
basic-auth, body from string / file / `-d @body.json`, `--cert` /
`--key` mTLS, HTTP/2 (`--http2`), HTTPS verify-skip
(`--insecure`), per-request body templating from a CSV / JSONL
`-B body.tmpl` so a 100k-RPS test doesn't pound the same payload,
and a `--duration` / `--count` / `--listen` triad for "run for 5
minutes" vs "send exactly N requests" vs "expose the metrics
endpoint and let Prometheus scrape it". On exit it dumps a
final-summary table plus optional JSON (`-o json`) suitable for
diffing against a previous run as a perf regression gate in CI.

## Why it's interesting
Different shape from `ab` (Apache Bench: single-threaded view of a
multi-threaded run, no live UI, latency histogram is summary-only,
HTTP/1.1 only), from `wrk` / `wrk2` (excellent throughput numbers
but Lua scripting + no live web UI + no per-status grouping out of
the box — you build the dashboard yourself), from
[`oha`](../oha/) (Rust, also has a TUI, also great — `plow` is
the pick when you specifically want the *web UI* served on a port
because the audience is more than one person watching the
terminal), from `hey` (simple, one-shot, no live view, no live web
UI, fewer percentiles), from k6 / Locust / Gatling / JMeter
(scripted scenario runners with sessions / think-time / data-
driven users — heavyweight; `plow` is the *one URL, full
throttle, show me the histogram* primitive), and from `vegeta`
(attack file → results file → render-after-the-fact pipeline —
plow's value is the *during-the-run* visibility). Pick plow when
the question is "what's the p99 of `POST /v1/chat/completions`
on my LLM gateway at 200 concurrent connections, and can my coworker
watch the chart from their browser without me sharing my screen"
and the answer needs to fit in one binary you `scp`'d to a
load-generator EC2 instance ten seconds ago. Do **not** pick it
for multi-step user journeys with sessions and assertions (use
k6 / Locust), for distributed load from many generator nodes (use
k6 cloud / Locust workers / Vegeta + a dispatcher), or when the
ask is reproducible CI perf regression checks rather than
interactive exploration (use [`oha`](../oha/) `--no-tui -j` or
vegeta into a JSON diff).

## Niche category
HTTP load generator with simultaneous live terminal table +
self-served web-UI charts, single Go binary, single URL focus.

## Repo
https://github.com/six-ddc/plow

## Version pinned
`v1.4.0` (latest tagged release as of 2026-05-01)

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`
  (https://github.com/six-ddc/plow/blob/master/LICENSE)

## Install
```sh
# Homebrew (macOS / Linux)
brew install plow

# Go install
go install github.com/six-ddc/plow@latest

# Docker
docker run --rm ghcr.io/six-ddc/plow https://example.com -c 50 -d 30s

# Prebuilt binary
curl -L https://github.com/six-ddc/plow/releases/download/v1.4.0/plow_1.4.0_macos_arm64.tar.gz \
  | tar xz && sudo mv plow /usr/local/bin/
```

## Usage examples
```sh
# 50 concurrent workers for 30s against a URL, live TUI table
plow https://example.com/api/v1/health -c 50 -d 30s

# Open the live web UI on http://127.0.0.1:18888 for the same run
plow https://example.com -c 100 -d 60s --listen :18888

# Fixed RPS instead of fixed concurrency: 500 req/s for 2 minutes
plow https://api.example.com/v1/embed -q 500 -d 2m

# POST a JSON body, mTLS, HTTP/2
plow https://gw.internal/v1/chat/completions \
  -m POST -T application/json -d @body.json \
  --cert client.pem --key client.key --http2 -c 200 -d 5m

# Body templating: each request gets a different prompt from a JSONL file
plow https://gw.internal/v1/chat -m POST -T application/json \
  -B prompts.jsonl -c 50 -d 1m

# Emit final results as JSON for a CI perf-regression diff
plow https://example.com -c 50 -d 10s -o json > run.json
```

## Date added
2026-05-01
