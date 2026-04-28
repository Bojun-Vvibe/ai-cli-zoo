# stern

- **Repo:** https://github.com/stern/stern
- **Version:** v1.33.1 (latest stable, 2026)
- **License:** Apache-2.0 ([LICENSE](https://github.com/stern/stern/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install stern` · `go install github.com/stern/stern@latest` · `pacman -S stern` · prebuilt binaries on the GitHub release page · binary name is `stern`

## What it does

`stern` is a **multi-pod, multi-container log tailer** for
Kubernetes. Where `kubectl logs -f` follows exactly one container
in one pod, `stern <regex>` follows every container in every pod
whose name matches the regex, across one or more namespaces, with
each line color-tagged by pod and container so you can read a
multi-replica service's interleaved output as a single stream. It
reconnects automatically when pods are killed and rescheduled, so
a rolling-update or HPA scale-up does not interrupt your tail.

Beyond the basic match-and-follow, it handles the operational
plumbing you actually want from a log tail: `--since 5m` to anchor
the start, `--tail 100` per container, `--include` / `--exclude`
regexes on the log lines themselves, `--container` to scope by
container name inside each pod, JSON output for piping into
`jq` / `gron` / `lnav`, and `--template` for custom Go-template
formatting (e.g. squash to one-line-per-event for ingest).
Auth is whatever your `~/.kube/config` already does; no operator,
no in-cluster install, no log aggregator required.

## When to pick it / when not to

Reach for `stern` when you are debugging a service with multiple
replicas and need to see every replica's output at once
(canary vs stable, leader vs follower, before vs after a rolling
restart). It is the right tool when you do not have a centralized
log stack — or when you do, but the lag/sampling/retention of
that stack is hiding the event you care about. It is also the
fastest way to confirm "is the new pod actually serving traffic
yet?" during a rollout, since you can `stern <deploy>-` from the
moment `kubectl rollout status` finishes.

Skip it when you already have a working centralized log UI
(Loki+Grafana, Elasticsearch+Kibana, Datadog, OTel collector
fan-in) and the question is historical — `stern` is a live tail,
not a search engine. Skip it on very large clusters where matching
a broad regex would open hundreds of concurrent log streams and
hit the API server hard. For single-pod follow with no replicas,
plain `kubectl logs -f` is one fewer dependency.

## Why it matters in an AI-native workflow

Coding agents that operate against a Kubernetes dev cluster —
deploying a build, running an integration test, mutating a CRD —
need a way to read what the cluster is actually doing in response.
Piping `stern -n <ns> <regex> --since 1m --no-follow` into the
agent's context after each apply gives a deterministic, bounded
log slice tied to the action just taken, rather than asking the
agent to invent the right `kubectl logs` invocation across
N replicas and M containers. The regex+namespace scoping also
keeps the log payload small enough to fit a context window, which
plain `kubectl logs --all-containers --prefix` does not.

## Example invocations

```bash
# Tail every pod whose name matches "api" in the current namespace
stern api

# Tail across all namespaces
stern --all-namespaces api

# Only the last 5 minutes, then follow
stern --since 5m api

# A specific container inside multi-container pods
stern api --container envoy

# Filter the log stream itself (server-side regex on the line)
stern api --include 'level=error|panic'

# JSON output for piping into jq
stern api --output json | jq 'select(.message | contains("timeout"))'

# One-shot snapshot, no follow (useful in scripts and agent loops)
stern api --since 1m --no-follow --tail 200

# Custom template: one compact line per event
stern api --template '{{.PodName}} {{.Message}}{{"\n"}}'
```

## Alternatives in this catalog

- [`k9s`](../k9s/) — full TUI for cluster ops; its built-in log
  view covers single-pod cases, stern wins on multi-pod regex.
- [`lnav`](../lnav/) — log navigator with format auto-detection;
  pipe `stern --output raw` into lnav for a searchable scrollback.
- [`tailspin`](../tailspin/) — colorizes any log stream by content;
  `stern api | tailspin` adds severity highlighting on top of
  stern's per-pod coloring.
