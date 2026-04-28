# httpstat

- **Repo:** https://github.com/davecheney/httpstat
- **Version:** v1.2.1 (latest stable, January 2025)
- **License:** MIT ([LICENSE](https://github.com/davecheney/httpstat/blob/master/LICENSE))
- **Language:** Go (single static binary, no runtime deps)
- **Install:** `brew install httpstat` · `go install github.com/davecheney/httpstat@latest` · `pacman -S httpstat` · static binaries on the GitHub release page · binary name is `httpstat`

## What it does

`httpstat` runs a single HTTP(S) request and renders an ASCII
**timeline** of where the wall-clock time actually went: DNS lookup,
TCP handshake, TLS handshake, server processing (the time between
your last byte sent and the server's first byte back, i.e.
**TTFB**), and content transfer. Each phase is shown both as an
absolute duration and as a cumulative offset, so you can see at a
glance whether a slow request is slow because of resolution, the
handshake, the backend, or the body bytes on the wire. Headers and
body are still printed normally above the timeline, so it doubles
as a debugging `curl` for one-off requests. Under the hood it uses
Go's `net/http/httptrace` hooks — there is no proxy, no MITM, and
no extra round trip; the numbers are the request you'd otherwise
have run, just instrumented.

## When to pick it / when not to

Pick `httpstat` the moment someone says "the API feels slow" and
you don't yet know which hop owns the latency. It is the right
tool for proving that a 600 ms request is 580 ms of TLS handshake
on a cold connection (so the fix is keep-alive, not a backend
optimisation), for showing that a CDN edge is fast but the origin
TTFB is the problem, for checking whether DNS resolution is the
tail latency on a misconfigured resolver, and for sanity-checking
that HTTP/2 multiplexing is actually saving you the per-request
handshake cost. It is also a perfect "first response" diagnostic
to attach to a perf bug report — one screenshot tells reviewers
where to look.

Skip it for **load testing** — it runs exactly one request; reach
for [`oha`](../oha/), `wrk`, `vegeta`, or `k6` when you need
percentiles. Skip it for **scripted multi-step API tests with
assertions** — that's [`hurl`](../hurl/). Skip it for
**continuous probing** — wrap [`curl -w`](https://curl.se/docs/manpage.html#-w)
with a custom format string and ship to Prometheus / Datadog
instead, since `httpstat`'s output is meant for human eyeballs.
Skip it for **WebSocket / gRPC** — the timeline model assumes a
single request/response cycle.

## Why it matters in an AI-native workflow

Latency bugs are the worst category for an LLM agent to debug
blind: the symptom ("slow") is one word, the surface is huge
(DNS, TCP, TLS, app, payload), and the wrong fix wastes a round
of edits. `httpstat`'s output is **structured, deterministic,
and bounded** — five named phases, each with a number — which
makes it trivial for an agent to parse, attribute the dominant
phase, and pick a corrective action (enable keep-alive, switch
DNS, add a connection pool, gzip the response) on the first try
instead of the third. Drop the output into the conversation and
the model has all the evidence it needs in ~15 lines.

## Example invocations

```bash
# Basic timeline for a GET
httpstat https://api.github.com/

# POST a JSON body (any curl-style flag is forwarded)
httpstat -X POST https://api.example.com/v1/items \
  -H 'Content-Type: application/json' \
  -d '{"name":"alpha"}'

# Force HTTP/2 to compare handshake cost vs HTTP/1.1
httpstat --http2 https://www.cloudflare.com/

# Save the response body to a file while still printing the timeline
HTTPSTAT_SAVE_BODY=true httpstat https://example.com/large.json
# body lands in /tmp/httpstat-body.<pid>

# Show TLS / cert details inline (uses curl's -v under the hood)
HTTPSTAT_SHOW_BODY=false httpstat -v https://expired.badssl.com/

# Compare cold vs warm connection: run twice, second one skips DNS+TCP+TLS
httpstat https://api.example.com/health
httpstat https://api.example.com/health   # reuse not automatic — see keep-alive note
```
