# humanlog

> **Stdin filter that turns JSON / logfmt structured logs
> into human-readable, colorized lines** — drop-in pipe for
> `kubectl logs`, `docker logs`, `journalctl -o json`,
> `zap` / `logrus` / `slog` output. Pinned to **v0.7.8**
> ([LICENSE.md](https://github.com/humanlogio/humanlog/blob/master/LICENSE.md),
> Apache-2.0).

Source: <https://github.com/humanlogio/humanlog>

## TL;DR

`humanlog` is the single binary that fixes the daily papercut
of tailing a structured-logging service in development: the
producer emits one JSON object per line (`{"ts":"…","level":
"info","msg":"…","trace_id":"…"}`) which is great for Loki /
Datadog and unreadable for human eyes. Pipe it through
`humanlog` and you get a `bunyan`-style line — timestamp
relative-formatted, level color-coded, message in white,
remaining keys laid out as `key=value` pairs at the end —
without changing the producer. It auto-detects JSON vs
logfmt vs zap-development format per-line, parses every
common timestamp dialect (RFC3339, Unix seconds, Unix
milliseconds, `2006-01-02 15:04:05`), and falls through any
non-structured line unchanged so a partial migration (some
services structured, some not) still works through one pipe.

## Install

```bash
# Homebrew (macOS / Linux)
brew install humanlog

# Go install (any platform with a Go toolchain)
go install github.com/humanlogio/humanlog/cmd/humanlog@v0.7.8

# Pre-built binary (Linux / macOS, amd64 / arm64)
curl -sSL https://humanlog.io/install.sh | sh

# Verify
humanlog --version    # 0.7.8
```

## Example usage

```bash
# 1. Tail a Kubernetes pod's structured logs in human form
kubectl logs -f deploy/api | humanlog

# 2. Same trick for docker compose
docker compose logs -f --no-color api worker | humanlog

# 3. systemd journal — JSON out of journalctl, pretty in
journalctl -u myservice.service -f -o json | humanlog

# 4. Filter to warnings and above, keep only request_id + msg
kubectl logs -f deploy/api \
  | humanlog --truncate=false \
             --skip="trace_id,span_id,caller" \
             --keep="request_id,msg"

# 5. Persistent default: alias once, forget forever
echo 'alias hl=humanlog' >> ~/.zshrc
kubectl logs -f deploy/api | hl
```

## Why this lives in the zoo

Every modern service emits structured logs and every modern
developer reads them with their eyes during local debugging.
The producer-side fix ("just log pretty in dev") loses the
structure your aggregator needs; the aggregator-side fix
("only read logs in Loki / Datadog") is overkill for a
two-minute `kubectl logs -f`. `humanlog` is the
consumer-side seam that lets the same JSON line feed Loki
*and* read like `bunyan` did in 2014, with one binary you
chain after `kubectl` / `docker` / `journalctl` and never
think about again. Pairs naturally with [`lnav`](../lnav/)
(which is the *navigator* for log files at rest) — `humanlog`
is the *prettifier* for the live tail.
