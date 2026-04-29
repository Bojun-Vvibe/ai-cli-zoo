# vegeta

> **A versatile HTTP load-testing tool and Go library — drives a
> sustained, *constant request rate* attack against an HTTP target,
> emits a binary results stream, and reports latency percentiles +
> success-rate + a plot you can paste into a PR.** Pinned to **v12.13.0**,
> [LICENSE](https://github.com/tsenart/vegeta/blob/master/LICENSE),
> MIT.

Source: <https://github.com/tsenart/vegeta>

## TL;DR

`vegeta` is the load generator you reach for when you do not want
to think about whether your numbers are honest. Unlike `ab` /
`wrk` / `siege` (which fix concurrency and let throughput drift
with response latency), `vegeta` fixes **request rate** — `-rate=200`
means 200 req/s arrive at the target whether the server is healthy
or melting, which is the only model that produces a comparable
latency-vs-throughput curve across runs. The CLI is a small set
of orthogonal subcommands (`attack`, `encode`, `report`, `plot`,
`dump`) that pipe into each other: `vegeta attack -rate=200
-duration=60s -targets=t.txt | tee r.bin | vegeta report` runs
the load and prints `Latencies [min, mean, 50, 90, 95, 99, max]`
+ success ratio + bytes-in / bytes-out, and the same `r.bin` can
be re-reported as JSON, plotted as an interactive HTML chart, or
dumped to CSV without re-running the attack. Targets are a plain
text file (`POST https://api/v1/foo\n@body.json`) so curl-shaped
requests translate one-to-one.

## Install

```bash
# Homebrew (macOS / Linux)
brew install vegeta

# Go (any platform with a Go toolchain)
go install github.com/tsenart/vegeta/v12@latest

# Pre-built binary
curl -L "https://github.com/tsenart/vegeta/releases/download/v12.13.0/vegeta_12.13.0_$(uname -s | tr A-Z a-z)_amd64.tar.gz" \
  | tar xz vegeta && sudo mv vegeta /usr/local/bin/

# Docker
docker run --rm -i peterevans/vegeta vegeta -version

# Linux package managers
# Arch (AUR):  yay -S vegeta
# Nix:         nix-env -iA nixpkgs.vegeta

# verify
vegeta -version    # Version: 12.13.0
```

## License

MIT — see
[LICENSE](https://github.com/tsenart/vegeta/blob/master/LICENSE).
Permissive, no copyleft on output, redistribute the binary freely.

## One Concrete Example

```bash
# 1. simplest: 50 req/s for 30s against one URL
echo "GET https://httpbin.org/get" | vegeta attack -rate=50 -duration=30s | vegeta report

# 2. multiple targets, mixed methods, with bodies + headers
cat > targets.txt <<'EOF'
GET https://api.example.com/v1/health

POST https://api.example.com/v1/items
Content-Type: application/json
@body.json

DELETE https://api.example.com/v1/items/42
Authorization: Bearer ${TOKEN}
EOF

# 3. run the attack, save raw results, report + plot
vegeta attack -targets=targets.txt -rate=200 -duration=60s -timeout=5s > results.bin
vegeta report -type=text                results.bin
vegeta report -type=json                results.bin > report.json
vegeta report -type='hist[0,2ms,5ms,10ms,25ms,50ms,100ms,250ms,500ms,1s]' results.bin
vegeta plot   -title="api v1 @ 200 rps" results.bin > plot.html

# 4. step / ramp load (compose multiple attacks)
( vegeta attack -rate=50  -duration=30s -targets=t.txt
  vegeta attack -rate=200 -duration=30s -targets=t.txt
  vegeta attack -rate=500 -duration=30s -targets=t.txt
) | vegeta report

# 5. CI gate: fail the build if p99 > 250 ms or success < 99.5 %
vegeta attack -rate=100 -duration=30s -targets=t.txt \
  | vegeta report -type=json \
  | jq -e '.latencies."99th" < 250000000 and .success > 0.995'

# 6. HTTP/2, custom resolver, client cert, follow redirects off
vegeta attack -http2 -resolvers=1.1.1.1 -cert=client.pem -key=client.key \
              -redirects=-1 -rate=100 -duration=30s -targets=t.txt > r.bin
```

## Niche It Fills

**Constant-rate, reproducible HTTP load with honest latency
percentiles, no warmup-tax dance, no JVM, no DSL.** The space is
crowded — `wrk` / `wrk2`, `hey`, `bombardier`, `k6`, `locust`,
`gatling`, `jmeter` — but `vegeta` is the right answer when you
want a single static Go binary, a plain-text target file (no Lua,
no JS, no Python, no XML), and the *open-model* (constant-rate)
loading that produces apples-to-apples latency curves you can
paste into a PR. The binary results stream means a single attack
can be re-reported a dozen ways without re-loading the server,
which matters when the target is a paid third-party API or a
shared staging cluster.

## Why use it

Three things `vegeta` does that `ab` / `wrk` / `hey` do not:

1. **Open-model constant-rate loading.** `-rate=200` means 200
   requests *arrive* at the target per second, scheduled by the
   client clock, *not* "200 in flight at all times". This is the
   only model that does not suffer from
   [coordinated omission](https://www.scylladb.com/2021/04/22/on-coordinated-omission/):
   when the server slows down, the queue grows and the latency
   distribution reflects what real users would see, instead of
   silently throttling the offered load to whatever the server
   can handle. `wrk2` is the only other widely-used tool that
   gets this right.
2. **Compose-friendly Unix pipeline.** `attack` writes a binary
   results stream to stdout; `encode` / `report` / `plot` /
   `dump` read it from stdin. The same run can produce a text
   summary, a JSON report for CI gating, an HTML latency plot,
   and a CSV for post-hoc analysis without re-loading the target.
   Multiple `attack` invocations can be concatenated to build
   step / ramp / spike profiles without a separate scripting
   layer (`( vegeta attack ...; vegeta attack ...) | vegeta report`).
3. **Library + CLI parity.** The same Go package
   (`github.com/tsenart/vegeta/v12/lib`) that powers the CLI is
   importable, so a Go integration test can drive load
   programmatically and assert on the `Metrics` struct directly,
   sharing the target file with the standalone CLI run.

For an LLM-CLI workflow, `vegeta` is the right tool to load-test
an LLM-served HTTP endpoint (your own FastAPI wrapper, a self-hosted
inference server, an OpenAI-compatible proxy) where you want to
know the *real* p95 / p99 at a given offered rate, without writing
JS scenario code. Pair with `jq` on `vegeta report -type=json`
to feed the numbers back into a perf-regression gate.

## Vs Already Cataloged

- **Vs [`k6`](../k6/):** different tier of tool. `k6` is a full
  load-testing platform with a JavaScript scenario DSL, browser
  recording, thresholds, scenarios, executors, and a Grafana
  Cloud SaaS — pick `k6` when "scenario" (login → browse →
  add-to-cart → checkout) is the unit of work. `vegeta` is the
  right answer when the unit is "fire request X at rate Y" and
  you want a single binary + a text target file with no
  JavaScript runtime.
- **Vs [`oha`](../oha/):** both are Go / Rust single-binary HTTP
  load testers. `oha` is `hey`-shaped (closed-model concurrency,
  beautiful real-time TUI, `-z 30s -c 50`) — better for
  interactive "watch the requests fly" exploration. `vegeta` is
  open-model (constant-rate) with binary results streams — better
  for reproducible perf gates and side-by-side runs.
- **Vs [`grpcurl`](../grpcurl/):** orthogonal. `grpcurl` is a
  *protocol-aware client* for gRPC (reflection, descriptors, ad-hoc
  calls); `vegeta` is HTTP/1.1 + HTTP/2 load gen. Use `grpcurl`
  to *exercise* a gRPC API, `vegeta` (or `ghz`) to *load* it.
- **Vs [`xh`](../xh/) / [`httpie`](../httpie/):** also orthogonal.
  Those are interactive HTTP clients (request-shaping, JSON
  ergonomics); `vegeta` is for sustained load, not for crafting
  one-off requests.

## Caveats

- **HTTP/HTTPS only (HTTP/1.1 + HTTP/2).** No gRPC, no WebSocket,
  no MQTT, no raw TCP. For gRPC use [`ghz`](https://ghz.sh/);
  for WebSocket use [`websocat`](../websocat/) + a custom
  generator or `k6 ws`.
- **Single-machine load is bounded by the client kernel.** A
  single `vegeta` process can sustain ~10–50 k req/s on a
  modern Linux box with `ulimit -n` raised and `net.ipv4.*`
  tuned, but past that you need to shard across hosts (the docs
  show a `vegeta attack` over SSH fan-out pattern). For
  hundreds-of-thousands req/s you want a coordinated cluster
  (`k6` Cloud, `gatling-enterprise`, or roll your own).
- **Open-model loading can DoS your target unintentionally.**
  Because `-rate=N` is what arrives at the server regardless of
  responsiveness, pointing `vegeta` at a small staging cluster
  with `-rate=10000` will queue up file descriptors and
  in-flight requests faster than the box can respond. Always
  pair with `-max-workers` and `-timeout` in unfamiliar environments.
- **Targets file is line-oriented and brittle.** Bodies are
  loaded via `@filename` (filesystem-relative); template
  variables across requests are not built in. For dynamic
  per-request payloads (e.g. signed JWTs, randomised IDs),
  generate the targets file ahead of time with a script or use
  `vegeta` as a library and supply a custom `Targeter`.
- **Results file is binary and version-sensitive.** A `.bin` from
  v9 may not decode in v12; pin the binary version in CI and
  re-run the attack rather than archive the binary stream
  long-term — archive the JSON report instead.
