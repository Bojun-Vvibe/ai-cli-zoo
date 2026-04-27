# oha

> **HTTP load generator with a real-time TUI** — a Rust binary
> that fires N concurrent workers at a target URL, draws a live
> histogram of latency + a per-second RPS / status-code chart in
> a `tui-rs` interface while the run is in flight, and prints a
> classic `wrk`-style summary (percentiles p50 / p95 / p99 /
> p99.9, status-code histogram, error breakdown, response-size
> totals) at the end. Speaks HTTP/1.1, HTTP/2, and HTTP/3 (QUIC)
> with the same flag surface. Pinned to **v1.14.0** (commit
> `9766ca2b725b69b7606b99861d01c779b4aeb65d`,
> [LICENSE](https://github.com/hatoo/oha/blob/master/LICENSE),
> MIT).

Source: <https://github.com/hatoo/oha>

## TL;DR

`oha` is what you reach for when you want `wrk`'s numbers
without `wrk`'s Lua scripts and without `ab`'s 1990s output —
and you want to *see the run happening* instead of staring at a
blank terminal for 30 seconds. Same `-c <connections>` /
`-z <duration>` / `-n <requests>` knobs as the classics, plus a
live TUI that catches "the server died at second 12" while it is
still happening, plus first-class HTTP/2 and HTTP/3 so you can
benchmark a modern reverse proxy without swapping tools.

## Install

```bash
# Homebrew (macOS / Linux)
brew install oha

# Cargo
cargo install --locked oha

# Linux package managers
# Arch:    pacman -S oha
# Nix:     nix-env -iA nixpkgs.oha
# Alpine:  apk add oha

# from a release tarball (any OS)
curl -Lo oha.tar.gz "https://github.com/hatoo/oha/releases/download/v1.14.0/oha-v1.14.0-aarch64-apple-darwin.tar.gz"
tar xf oha.tar.gz
sudo install oha /usr/local/bin/

# verify
oha --version    # oha 1.14.0
```

`oha` runs anywhere a Rust + rustls binary runs; no system
libcurl, no Lua, no Python.

## License

MIT — see [LICENSE](https://github.com/hatoo/oha/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. classic 50-connection / 10-second smoke test (TUI on by default)
oha -c 50 -z 10s http://localhost:8080/health

# 2. fixed request count instead of duration, no TUI (CI-friendly)
oha --no-tui -n 10000 -c 100 http://localhost:8080/api/v1/items

# 3. POST a JSON body
oha -c 20 -z 30s \
  -m POST \
  -H 'Content-Type: application/json' \
  -d '{"name":"alice","age":30}' \
  http://localhost:8080/api/v1/users

# 4. force HTTP/2 (h2c over plaintext or h2 over TLS)
oha --http2 -c 100 -z 30s https://api.example.com/v1/ping

# 5. force HTTP/3 (QUIC) — for benchmarking a modern edge proxy
oha --http-version 3 -c 100 -z 30s https://h3.example.com/

# 6. machine-readable JSON summary (for diff vs a baseline)
oha --no-tui -j -n 5000 -c 50 http://localhost:8080/ > run.json
jq '.summary.requestsPerSec, .latencyPercentiles' run.json

# 7. cap the request rate (don't melt the box, find the latency floor)
oha -q 200 -c 50 -z 60s http://localhost:8080/  # 200 RPS ceiling

# 8. burst test from a list of URLs
oha --urls-from-file urls.txt -c 100 -z 30s
```

## Niche It Fills

**A modern, watchable, scriptable HTTP load tester.** The space
divides into "old school CLI" (`ab`, `wrk`, `wrk2`, `hey`) and
"big distributed harness" (`k6`, `vegeta`, `gatling`, `locust`).
The first group's output is a wall of text printed *after* the
run; the second group is a JS / Lua / Python config investment.
`oha` lives in the gap: the same shell-flag UX as `hey` /
`wrk`, plus a live `tui-rs` view of latency + RPS + status codes
*during* the run, plus modern protocol coverage (HTTP/2, HTTP/3
out of the box), plus a `--no-tui -j` mode that drops cleanly
into CI and trend-tracking pipelines.

## Why use it

Three things `oha` does that `ab` / `wrk` / `hey` do not, that
pay back the install:

1. **Live TUI while the run is in flight.** A real-time
   histogram of response latency plus a per-second RPS /
   status-code chart catches "the 95th tail is exploding right
   now" or "all responses turned 503 at second 8" *during* the
   run, not on the post-mortem printout. For interactive tuning
   ("now I'll bump the connection pool, rerun, watch the
   histogram shift") this collapses the iteration loop.
2. **HTTP/2 and HTTP/3 with one flag.** `--http2` and
   `--http-version 3` are first-class. `wrk` is HTTP/1.1 only;
   `hey` is HTTP/1.1 only; benchmarking a modern Envoy / NGINX /
   Caddy / Cloudflare front door without `oha` means installing
   a second tool.
3. **`--no-tui -j` is a clean CI primitive.** A single JSON
   document with `summary.requestsPerSec`, latency percentiles,
   status-code counts, and per-error tallies that you can `jq`
   into a regression check ("p99 must be < 200 ms," "error rate
   must be < 0.1%"). `wrk` requires Lua + custom stdout parsing
   to get the same.

## Vs Already Cataloged

- **Vs [`xh`](../xh/):** orthogonal — `xh` is a *one-shot* HTTP
  client (the `httpie`-shaped `xh GET ...` ergonomics for ad-hoc
  debugging), `oha` is a *load generator* that fires the same
  request thousands of times in parallel and reports
  distributions. Use `xh` to confirm the request shape works,
  then hand the same URL + body off to `oha` for the load test.
- **Vs [`hyperfine`](../hyperfine/):** orthogonal — `hyperfine`
  benchmarks *commands* (statistical timing of `cargo build` vs
  `cargo build --release`), `oha` benchmarks *HTTP endpoints*.
  Different units, different methodology. Reach for `hyperfine`
  for "is this CLI faster than that one"; reach for `oha` for
  "how many RPS can this service take before the p99 climbs."
- **Vs `wrk` / `wrk2` (not cataloged):** the classical peer.
  `wrk` is mature, scriptable in Lua, and the de-facto baseline
  in performance papers. `oha` matches the headline numbers and
  adds the TUI, HTTP/2, HTTP/3, and a clean JSON output mode at
  the cost of Lua-level scripting power. Pick `wrk` for the
  Lua scenarios you have already written; pick `oha` for any
  new project.
- **Vs `hey` (not cataloged):** the closest spiritual peer in
  feel — single binary, shell-flag UX, simple summary. `oha`
  improves on `hey` with the live TUI, HTTP/2, HTTP/3, and JSON
  summary; `hey` is HTTP/1.1-only and prints only after the
  run. Migrate `hey` users by translating flags 1:1.
- **Vs `ab` (not cataloged):** historical baseline shipped with
  Apache. Useful only because it is everywhere; `oha` wins on
  every dimension that is not "is preinstalled on this box."
- **Vs `k6` / `vegeta` / `locust` (not cataloged):** different
  weight class — those are full load-test frameworks with
  scripted scenarios, weighted endpoints, ramp profiles, and
  distributed runners. `oha` is the right answer for *single
  endpoint, single shape, single run* benchmarks; reach for the
  frameworks when you need a 30-step scripted user journey or a
  multi-machine fan-out.

## Caveats

- **Single-machine load only.** No distributed worker mode; the
  ceiling is the box you run it on (kernel, file descriptors,
  ephemeral port range). For multi-machine load reach for `k6`
  or `vegeta`.
- **TUI hides cost on huge runs.** Rendering the live charts
  costs CPU; on a maxed-out box the TUI itself starts to compete
  with the load generator. Use `--no-tui` for the highest
  achievable RPS or for headless CI.
- **HTTP/3 is a moving target.** `oha`'s HTTP/3 support depends
  on the underlying `quinn` / `h3` versions in the build; some
  servers / load balancers still gate QUIC on specific ALPN
  identifiers. If `--http-version 3` errors out, fall back to
  HTTP/2 to confirm the rest of the path.
- **No request scripting.** Each worker fires the same request
  (URL / method / headers / body). For per-request variation
  (e.g., random user IDs, signed requests) build the URL list
  externally and use `--urls-from-file`, or step up to `k6` /
  `vegeta` for true scenario scripting.
- **TLS verification is on by default; turn it off explicitly
  for self-signed dev servers.** Use `--insecure` (mirrors
  `curl -k`) and treat that as a smell — it should not be in
  any committed CI script that targets production.
- **Statistics are point-in-time, not warm-up-aware.** The
  summary covers the full run; if you want "steady-state only"
  numbers, drop the first few seconds with `-z` plus a separate
  warm-up pass, or post-process the per-request log (when
  enabled).
