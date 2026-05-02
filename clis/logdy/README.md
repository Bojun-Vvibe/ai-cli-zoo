# logdy

> **A realtime log viewer with a built-in web UI** — a single
> Go binary that takes anything you can `tail -f` (file, stdout
> of a long-running process, multiple sources merged) and serves
> it on `localhost:8080` as a structured, filterable, JSON-aware
> stream you scroll in a browser instead of fighting `less +F`.
> Pinned to **v0.17.1**
> ([LICENSE](https://github.com/logdyhq/logdy-core/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/logdyhq/logdy-core>

## TL;DR

`logdy` sits in the awkward gap between `tail -f | grep`
(fine until you need to scroll back through 50k lines without
losing your filter) and Loki/Grafana (great, but you don't want
to stand up a stack to debug one container). You pipe stdin in,
or point it at one or more files, and it spawns a local HTTP
server with a single-page UI that streams new lines over a
WebSocket, parses each line as JSON when possible, lets you
write a tiny TypeScript snippet to extract / rename / colorize
fields, and gives you persistent filters, multi-tail merging,
and follow-mode toggling that your terminal pager doesn't. The
log payload never leaves the box — there's no upload, no
account, no telemetry. Useful for: a noisy dev server you want
to read while you keep editing in the same terminal, a
multi-process foreman/overmind run where you actually want
per-process color and filter, an ad-hoc `kubectl logs -f` you
want to pause-and-search without losing the live tail.

## Install

```bash
# Homebrew (third-party tap, macOS / Linux)
brew install logdy

# Single-binary download (GitHub releases, ~45 MB)
curl -L -o logdy \
  https://github.com/logdyhq/logdy-core/releases/download/v0.17.1/logdy_darwin_arm64
chmod +x logdy && sudo mv logdy /usr/local/bin/

# Build from source (Go >= 1.21)
git clone --depth 1 --branch v0.17.1 \
  https://github.com/logdyhq/logdy-core.git
cd logdy-core && go build -o logdy
```

## Usage

```bash
# Pipe a noisy dev server through logdy, open browser on :8080
npm run dev | logdy

# Multi-tail two files, merged into one stream with source labels
logdy follow ./api.log ./worker.log

# Bind to a custom port and require a UI password (shareable over SSH tunnel)
logdy --port 8123 --ui-pass hunter2 -- node server.js
```

## Why it's interesting

Most "web UI for logs" projects either want to be a SaaS
(Logtail, Better Stack) or assume you've already adopted a
log shipper (Vector → Loki → Grafana). `logdy` is the rare
local-first, single-binary, zero-config option in that niche:
no daemon, no database, no agent. The browser UI is what
makes it stand out — JSON-line logs become a real table you
can sort and filter, and the inline TypeScript hook lets you
do `row.level === 'error' && row.svc === 'auth'` style
filtering without leaving the page. Compare to `tail -f` (no
structure), `lnav` (TUI, not shareable), `frontail` (older,
unmaintained, no JSON parsing), or shipping to Loki (way
more setup for a one-off debug session). The one real
trade-off is the binary is ~45 MB because it embeds the
compiled web UI; there's no `--no-ui` mode that strips it.
