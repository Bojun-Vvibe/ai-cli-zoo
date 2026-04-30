# wrk

- **Repo:** https://github.com/wg/wrk
- **Latest version:** 4.2.0 (tag `4.2.0`)
- **HEAD on `master`:** `a211dd5`
- **License:** Modified Apache 2.0 — [LICENSE](https://github.com/wg/wrk/blob/master/LICENSE)
- **Category:** network / HTTP load testing

A **modern HTTP benchmarking tool** that combines a multithreaded
event-loop core (epoll on Linux, kqueue on macOS / BSD) with an
embedded **LuaJIT** scripting layer. A single `wrk` process pins one
OS thread per `-t` worker, multiplexes thousands of `-c` keep-alive
connections per worker through non-blocking I/O, and reports the
latency distribution at p50 / p75 / p90 / p99 / p99.99 plus
requests-per-second and transfer-per-second over the configured
duration. The Lua `request()` / `response()` / `done()` hooks let you
shape per-request URL, headers, body, and post-run aggregation
without recompiling, so the same binary covers static-asset hammering,
parameterised REST replay, and custom report formats. The default
output is a one-screen text histogram designed to fit in a terminal
scroll-back next to the command that produced it — no separate UI,
no daemon, no persistent state.

## Install

```bash
# macOS
brew install wrk

# Debian / Ubuntu
apt-get install wrk

# Fedora / RHEL
dnf install wrk

# Arch
pacman -S wrk

# from source (requires make + a C compiler + OpenSSL headers)
git clone https://github.com/wg/wrk.git
cd wrk && make
sudo cp wrk /usr/local/bin/

# verify
wrk --version
```

## One usage example

```bash
# 12 threads, 400 connections, 30 seconds, latency histogram on:
wrk -t12 -c400 -d30s --latency https://example.com/api/health
```

Sample output:

```
Running 30s test @ https://example.com/api/health
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    18.42ms   12.30ms 312.40ms   88.21%
    Req/Sec     1.81k   223.10     2.91k    74.50%
  Latency Distribution
     50%   16.10ms
     75%   22.40ms
     90%   31.20ms
     99%   78.60ms
  650412 requests in 30.01s, 142.30MB read
Requests/sec:  21672.18
Transfer/sec:      4.74MB
```

## When to pick `wrk` (vs alternatives)

- Pick **`wrk`** when you want the *latency distribution* (p99 / p99.99)
  of a single endpoint under a fixed concurrency level, with the
  smallest possible client-side overhead (one binary, no runtime, no
  YAML). The Lua `request()` hook is the right knob when you need
  parameterised replay (rotating bearer tokens, randomised query
  strings) but do not want to learn a scenario DSL.
- Pick [`oha`](../oha/) when you want the same single-endpoint
  hammering loop *plus* a live TUI showing the histogram update in
  real time, or when you cannot install a C toolchain on the load
  generator (oha is a single static Rust binary).
- Pick [`vegeta`](../vegeta/) when the unit of work is a **mixed-URL
  attack file** at a constant *requests-per-second* rate (open-loop)
  rather than `wrk`'s **fixed-concurrency** loop (closed-loop), and
  you want the raw per-request results piped into `vegeta report` /
  `vegeta plot` for offline analysis.
- Pick [`k6`](../k6/) when the scenario is a *user journey* (login →
  browse → checkout) with stages, thresholds, checks, and
  CI-friendly pass/fail output — k6's JS scripting and built-in
  Prometheus / OTel exporters are made for that, where `wrk` is made
  for "how fast does this one URL go".
- Pick [`hyperfine`](../hyperfine/) for **command-line** benchmarking
  (`hyperfine 'cargo build'`); `wrk` is for HTTP servers, `hyperfine`
  is for processes.
