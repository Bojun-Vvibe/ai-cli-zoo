# ali

- **Repo:** https://github.com/nakabonne/ali
- **Version:** v0.8.0 (latest stable, January 2026)
- **License:** MIT ([LICENSE](https://github.com/nakabonne/ali/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install nakabonne/ali/ali` · `go install github.com/nakabonne/ali@latest` (untagged) · `apk add ali` · `pacman -S ali` · `apt install ./ali_*.deb` from the GitHub release page · static binaries on the GitHub release page · binary name is `ali`

## What it does

`ali` is a single-binary HTTP load generator that **plots its own
results in real time** inside the terminal. You point it at a URL, it
starts hammering the target at a configurable rate, and a termdash-based
TUI updates a latency chart, a percentile chart (p50/p90/p95/p99), a
bytes-in/out chart, and a histogram **while the attack is still
running** — you can press `l` / `h` to cycle charts and click-and-drag
to zoom into a region. Under the hood the attacker is built on top of
`vegeta`'s library API, so the request engine is the same battle-tested
Go HTTP load core (constant-rate workload, configurable connections,
HTTP/2 by default, custom resolvers, client certs, keep-alive control)
— `ali` is what you get when you take that engine and bolt a live
dashboard on top of it instead of dumping a histogram only after the
run finishes. Results can also be exported to a directory
(`--export-to ./results/`) for downstream analysis.

## When to pick it / when not to

Pick `ali` when you want to **watch** how a service degrades as you
push it — saturation point, when p99 starts walking away from p50, the
exact moment connection-pool exhaustion shows up as a latency cliff.
The live chart is genuinely useful for long runs (`--duration=0` for
infinite, or `--duration=10m` for a soak test) where you'd otherwise
be staring at a spinner. It's also a great teaching tool for showing
non-engineers what "tail latency" actually looks like.

Pick something else when you need:

- **CI assertions / SLO gates** — use [`vegeta`](https://github.com/tasschereau/vegeta)
  directly (or [`oha`](../oha/), [`plow`](../plow/),
  [`hey`](https://github.com/rakyll/hey)) so you can pipe the JSON
  report into a checker. `ali` is interactive-first; its `--export-to`
  is for offline review, not for `if p99 > 200ms then fail`.
- **Scripted multi-step scenarios** — login, then GET, then POST with
  capture — that's [`hurl`](../hurl/) or [`k6`](https://k6.io/)
  territory, not single-URL load.
- **Distributed load from many machines** — `ali` runs in one process
  on one host; for coordinated multi-region load, reach for `k6`,
  `locust`, or `vegeta` with `attack -duration=... | report` piped
  across a fleet.
- **WebSocket / gRPC / SSE** — `ali` is HTTP/1.1 + HTTP/2 only.

## Why it matters in an AI-native workflow

LLM-driven coding agents are increasingly asked to do performance
work: "this endpoint got slower after the last deploy, what changed".
The hard part of that loop is not generating the load — `curl` in a
`for` loop can do that — it's **noticing** the regression in the
shape of the latency distribution, not just the mean. `ali`'s live
percentile chart makes the regression visually obvious within seconds,
which means the agent (or the human pairing with it) can iterate on a
fix, re-run, and immediately see whether p99 came back to baseline,
without writing a stats harness. The export directory gives a
machine-readable artifact the agent can attach to a PR description as
evidence ("p99 went from 412 ms to 187 ms after this change").

The other reason it matters: a lot of "is this LLM gateway fast
enough" questions are really "what's the p99 at 50 rps with 4 KB
prompts" questions. `ali --rate=50 --duration=2m --body-file=prompt.json
--method=POST -H 'Content-Type: application/json' https://gw/v1/chat`
answers that in two minutes with a chart you can screenshot into a
design doc.

## Example invocations

```bash
# Default: 50 rps for 10 s, then press Enter to start the attack.
ali http://host.xz

# Sustained 500 rps for 5 minutes.
ali --rate=500 --duration=5m http://host.xz

# Infinite attack — Ctrl-C to stop. Useful for long soak / leak hunts.
ali --duration=0 --rate=100 http://host.xz

# POST with a JSON body from a file, custom header, HTTP/2 disabled.
ali --method=POST --body-file=./payload.json \
    --header='Content-Type: application/json' \
    --no-http2 --rate=200 --duration=2m \
    https://api.example.test/v1/predict

# Override DNS resolvers (useful when targeting a service mesh).
ali --resolvers=1.1.1.1:53,8.8.8.8:53 --rate=100 https://host.xz/health

# Export per-request results to a directory for later analysis.
ali --rate=100 --duration=1m --export-to=./run-2026-05-02 https://host.xz
```

## Caveats

- `go install github.com/nakabonne/ali@latest` deliberately downloads
  an untagged commit — the maintainer recommends Homebrew, MacPorts,
  apk, pacman, the `.deb` / `.rpm` from GitHub Releases, or the raw
  binary, in roughly that order. Use the release page if you want a
  reproducible version pin.
- The TUI needs a real TTY with mouse support for the click-and-drag
  zoom; over `tmux` you may need `set -g mouse on`.
- Because the engine is `vegeta`'s constant-rate model, `ali` issues
  requests on a fixed schedule regardless of in-flight latency — this
  is the right model for measuring a service's behavior under a known
  arrival rate, but it is not a closed-loop "wait for response then
  send next" workload. If you need closed-loop, that is a different
  tool.
- Be careful what you point it at. Default rate is 50 rps; an infinite
  duration at high rates against a third-party service is abusive and
  may get your IP blocked. Always test against your own infrastructure
  or a documented load-test environment.
