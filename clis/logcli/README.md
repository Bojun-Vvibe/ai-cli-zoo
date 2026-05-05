# logcli

> **The official command-line client for Grafana Loki —
> LogQL queries against a remote log backend, streamed
> to your terminal** — a single Go binary that turns
> a Loki `/api/v1/query_range` HTTP endpoint into the
> same kind of fluent CLI that `psql` is for Postgres
> or `redis-cli` is for Redis: ad-hoc LogQL one-liners
> from the shell, `--tail` for live streaming, label /
> series introspection, instant queries, range queries
> with `--from` / `--to`, JSON / raw output, statistics,
> and Prometheus-shaped metric queries derived from log
> aggregations. Pinned to **Loki v3.7.1** (commit
> `2c8fff222bab6813374b973ae0eb49043d3ed14e`,
> [LICENSE](https://github.com/grafana/loki/blob/main/LICENSE),
> AGPL-3.0).

Source: <https://github.com/grafana/loki/tree/main/cmd/logcli>

## TL;DR

`logcli` is the missing terminal piece of a Loki
deployment: where Grafana Explore is the GUI for ad-hoc
log queries, `logcli` is the CLI that lets you pipe the
same LogQL into `awk` / `jq` / `grep` / `wc` for the
case where the answer is "give me the last 1000 lines
matching this label set, raw, so I can run a sed on
them" rather than "render a chart". One binary, one
flag set (`--addr`, `--username`, `--password`,
`--bearer-token`, plus environment variables for all of
them) speaks to local Loki, hosted Grafana Cloud, or
any self-hosted Loki cluster behind an HTTPS proxy. The
seven verbs cover the entire useful surface: `query`
(range query, the default mode), `instant-query` (point
query), `labels` (list label names or values), `series`
(active series matching a matcher), `stats` (chunk +
ingester stats for a query), `volume` (per-stream byte
volume), and `--tail` (streaming follow, with an
optional `--delay-for` to absorb late-arriving logs).

## Install

```bash
# Homebrew (macOS / Linux)
brew install logcli

# Pre-built release binaries (Linux / macOS / Windows)
curl -Lo logcli.zip "https://github.com/grafana/loki/releases/download/v3.7.1/logcli-darwin-arm64.zip"
unzip logcli.zip
sudo install logcli-darwin-arm64 /usr/local/bin/logcli

# Docker (handy for CI runners that already have docker)
docker run --rm -e LOKI_ADDR="$LOKI_ADDR" grafana/logcli:3.7.1-amd64 query '{job="myapp"}'

# Build from source
git clone https://github.com/grafana/loki && cd loki
go build -o logcli ./cmd/logcli
sudo install logcli /usr/local/bin/

# verify
logcli --version    # logcli, version 3.7.1
```

Configuration is environment-variable-first:
`LOKI_ADDR=https://logs-prod-us-central1.grafana.net`,
`LOKI_USERNAME` / `LOKI_PASSWORD` (basic auth),
`LOKI_BEARER_TOKEN` (preferred for Grafana Cloud), and
`LOKI_ORG_ID` for multi-tenant clusters. A line in
`~/.zshrc` or a per-shell direnv file is enough; from
then on `logcli query '{job="x"}'` Just Works.

## License

AGPL-3.0 — see
[LICENSE](https://github.com/grafana/loki/blob/main/LICENSE).
The CLI binary itself is freely usable and pipeline-safe;
embedding `logcli` source in a closed product needs a
review of the AGPL network-use clause (it triggers when
you *modify* the source and serve it over a network,
not when you call the CLI from a script).

## One Concrete Example

```bash
# 1. last 1h of logs for one job, default tabular output
logcli query '{job="ingester"}'

# 2. last 24h, JSON output for jq pipelines
logcli query --since=24h --output=jsonl '{job="ingester"}' \
  | jq -r 'select(.line | contains("ERROR")) | .line'

# 3. live tail (Ctrl-C to stop) with regex filter inside LogQL
logcli query --tail '{job="api"} |~ "5\\d{2}"'

# 4. instant query — count of error lines per service in the last 5m
logcli instant-query \
  'sum by (service) (count_over_time({env="prod"} |= "ERROR" [5m]))'

# 5. discover what labels are available on this Loki tenant
logcli labels                           # list label names
logcli labels job                       # list values of `job`

# 6. find all active series for a matcher in the last 1h
logcli series '{env="prod", level="error"}' --since=1h

# 7. quantify how much volume a stream is producing (cost / cardinality)
logcli volume '{job=~"api|worker"}' --since=24h

# 8. pull a specific time window for an incident postmortem
logcli query --from="2026-04-30T13:00:00Z" \
              --to="2026-04-30T14:30:00Z" \
              --output=raw \
              '{job="api"} |= "panic"' > incident-stream.log

# 9. stats for a heavy query before you actually run it
logcli stats '{job="ingester"} |= "GC"' --since=6h
# → bytesProcessed, linesProcessed, chunksDownloaded, etc.
```

## Niche It Fills

**LogQL queries against a *remote* Loki cluster from a
terminal pipeline.** Local-file log viewers
([`lnav`](../lnav/), [`tspin`](../tspin/),
[`lazyjournal`](../lazyjournal/), [`frogmouth`](../frogmouth/),
[`toolong`](../toolong/)) read what's already on disk;
metric clients (`promtool query`,
[`promtool`](../promtool/)) speak PromQL against
Prometheus. `logcli` is the corresponding shell-native
verb for the third axis: aggregated logs at the cluster
level, queried by label selector and LogQL filter,
returned as text or JSON. Without it, the only way to
get a Loki query into a shell pipeline is to hit the
Loki HTTP API directly with `curl` and parse the JSON
envelope — `logcli` collapses that to one command.

## Why use it

Three things `logcli` does that ad-hoc `curl` against
the Loki API does not:

1. **LogQL ergonomics with `--tail` streaming.** The
   `query --tail` mode opens a websocket-like long-poll
   to Loki and streams matching lines as they arrive,
   with `--delay-for=Ns` to wait for late ingestion
   (the typical Loki write path is eventually
   consistent on the order of seconds). That's the
   `tail -f` for distributed logs that the Loki HTTP
   API does not directly expose to a `curl` user.
2. **Label / series / volume introspection as
   first-class verbs.** `logcli labels`, `logcli
   series`, and `logcli volume` reach the lesser-known
   `/loki/api/v1/labels`, `/series`, and `/index/volume`
   endpoints with sane flag defaults — the same
   discovery loop you'd run in Grafana Explore's label
   browser, but in shell pipelines for "is the
   cardinality on this label exploding?" CI gates.
3. **Stats and quiet-mode output for cost gates.**
   `logcli stats` returns `bytesProcessed` /
   `linesProcessed` / `chunksDownloaded` for a query
   before you actually run it, suitable for a CI gate
   that fails a PR if a new dashboard panel would
   process more than N GB. `--quiet` plus
   `--output=raw` gives you the matched lines and
   nothing else, ready for `wc -l` or
   `sort -u | head`.

For an LLM-CLI workflow doing incident triage, `logcli
query --since=1h --output=raw '{job="x"} |= "panic"'`
gives the agent the exact set of panic lines it needs
to reason about, in the same shell session as the rest
of its tool calls — no Grafana login, no copy-paste
from a browser, no API-token-handling code.

## Vs Already Cataloged

- **Vs [`lnav`](../lnav/) / [`tspin`](../tspin/) /
  [`lazyjournal`](../lazyjournal/) /
  [`frogmouth`](../frogmouth/) / [`toolong`](../toolong/)
  / [`humanlog`](../humanlog/):** orthogonal — those
  read **local files** (`/var/log/*`, `journalctl`,
  containers on the host); `logcli` queries a
  **remote Loki cluster** that aggregates logs from
  many hosts. Same verbs (filter, search, tail),
  different data plane. Use the local viewers to
  triage one box, use `logcli` to ask a question
  across the fleet.
- **Vs [`promtool`](../promtool/):** sibling — same
  Grafana / Prometheus ecosystem and similar CLI
  shape, different query language and backend.
  `promtool query` speaks PromQL to Prometheus;
  `logcli query` speaks LogQL to Loki. The two
  compose: `logcli instant-query 'sum by
  (service)(count_over_time({env="prod"} |= "ERROR"
  [5m]))'` produces a Prometheus-shaped vector that
  reads the same as the equivalent `promtool query
  instant ...` against an alerting metric.
- **Vs [`stern`](../stern/) / [`kail`](../kail/):**
  partial overlap on the live-tail axis, but those
  tail logs **directly from `kubectl`** (one or many
  pods at once). They stop working the moment a pod
  rotates out of the cluster's log buffer. `logcli`
  reads from Loki, which has retention measured in
  weeks-to-months, so it answers "what did the
  worker say at 03:14 last Tuesday" that `stern`
  cannot.
- **Vs [`vector`](../vector/) /
  [`fluent-bit`](../fluent-bit/):** different role —
  those are the **log shippers** that send data into
  Loki (or Elasticsearch, S3, etc.); `logcli` is the
  **query client** that reads it back out. A real
  deployment uses both: a shipper to ingest, `logcli`
  for ad-hoc queries from the terminal, and Grafana
  for dashboards.

## Caveats

- **Loki-only.** `logcli` speaks the Loki HTTP API
  exclusively — no Elasticsearch, no Splunk, no
  CloudWatch Logs. For a heterogeneous backend you
  need a separate per-source CLI (or a tool like
  `vector tap` to redirect logs into Loki first).
- **AGPL-3.0 affects modification, not invocation.**
  Calling `logcli` from a closed-source script or CI
  pipeline is fine; modifying the `logcli` source and
  exposing the modified binary as a network service
  triggers the AGPL source-disclosure clause. Most
  users hit only the "call from a script" path.
- **`--tail` latency depends on Loki write-path
  consistency.** Lines may arrive out of order or
  with a few seconds of lag versus the originating
  host clock; use `--delay-for=10s` (or longer)
  before computing rates over the streamed output.
- **Auth is environment-variable-first, no built-in
  credential helper.** Multi-tenant setups need
  `LOKI_USERNAME` + `LOKI_PASSWORD` (or
  `LOKI_BEARER_TOKEN`) in every shell that runs
  `logcli`. Pair with [`direnv`](../direnv/) or
  [`pizauth`](../pizauth/) for per-project token
  injection rather than baking tokens into rc files.
- **Heavy queries can time out at the gateway, not
  at `logcli`.** Loki's query frontend has its own
  default timeout (often 60s); `logcli --timeout=5m`
  sets only the client-side wait. For multi-day
  range queries, narrow the time window or use
  `--parallel-duration` / `--parallel-max-workers`
  to split the work.
