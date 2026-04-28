# curlie

- **Repo:** https://github.com/rs/curlie
- **Version:** v1.8.2 (latest stable, March 2025)
- **License:** MIT ([LICENSE](https://github.com/rs/curlie/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install curlie` · `go install github.com/rs/curlie@latest` · `pacman -S curlie` · `scoop install curlie` · static binaries on the GitHub release page · binary name is `curlie`

## What it does

`curlie` is a thin Go wrapper around the **real `curl` binary** that
gives you the front-end ergonomics of [HTTPie](../httpie/) — colorised
syntax-highlighted output, JSON pretty-printing, sane default headers,
the `key=value` / `key:=value` / `key==value` argument shorthand for
form fields, raw JSON values, and query parameters — while still
shelling out to `curl` for the actual transfer. The point is that
every `curl` flag you already know (`-L`, `--http2`, `--resolve`,
`--cert`, `-x`, `--data-binary @-`) keeps working unchanged, so you
can flip a one-liner from "exploratory" to "production-grade" without
relearning anything. Output is auto-detected: when stdout is a TTY you
get colorised headers and a pretty body; when piped you get raw bytes
suitable for `| jq`. Because the bottom layer is `curl`, you inherit
HTTP/2, HTTP/3, NTLM, client-cert, ALPN, and proxy behaviour exactly
as your system's `curl` ships them — there is no second HTTP stack to
debug.

## When to pick it / when not to

Pick `curlie` when you live in `curl` muscle memory but want JSON
APIs to *look* like JSON in your terminal, when you want HTTPie-style
`name=value` body building without giving up `curl`'s flag surface,
and when you're writing snippets you'll later paste into a shell
script or a CI job — `curlie` arguments map 1-to-1 to `curl`
arguments, so the transition is mechanical. It is the right tool
for poking at a REST or GraphQL endpoint while you're building it,
for capturing a working invocation and pasting it into a runbook,
and for any environment where the rest of the team already speaks
`curl` and you don't want to introduce a second mental model.

Skip it when you need **scripted multi-step flows with captures and
assertions** — reach for [`hurl`](../hurl/), which is designed for
that. Skip it when you need **load testing** ([`oha`](../oha/),
`wrk`, `vegeta`). Skip it for **WebSocket / SSE / gRPC** as a primary
protocol — `curl` got WebSocket support recently but `websocat` /
`grpcurl` are still better tools for those jobs. Skip it if you
genuinely prefer HTTPie's pure-Python implementation (richer plugin
ecosystem, built-in download progress UX, sessions stored as JSON
files) — `curlie` deliberately doesn't reimplement those because the
underlying `curl` already covers the network side.

## Why it matters in an AI-native workflow

LLM-driven coding agents constantly need to probe HTTP APIs: hit a
staging endpoint, check a webhook, verify an OAuth callback, read a
response body, pipe it to `jq`, then commit a fix. `curl`'s output
is a wall of ANSI-less text that the model has to re-tokenise and
re-parse on every iteration; `curlie` emits the same bytes but with
status, headers, and JSON structurally separated, which both makes
human review faster and lets the agent reliably grep `^HTTP/` or
extract a single header without guessing about whitespace. Because
flags are `curl`-compatible, the agent can write one invocation
during exploration and the same invocation lands in the committed
script — no translation step, no "works locally but not in CI".

## Example invocations

```bash
# GET with pretty-printed JSON body and colourised headers
curlie httpbin.org/get

# POST JSON: name=value -> string field, name:=value -> raw JSON
curlie POST httpbin.org/post \
  name=alice role:=\"admin\" tags:='["a","b"]' active:=true

# Query string params with == (curl's --data-urlencode equivalent)
curlie httpbin.org/get q==hello lang==en

# Custom header with name:value (HTTPie syntax)
curlie httpbin.org/headers X-Token:abc123 Accept:application/json

# Pipe a raw body in (falls through to curl --data-binary @-)
echo '{"k":1}' | curlie POST api.example.com/v1/items \
  Content-Type:application/json --data-binary @-

# Any curl flag still works: HTTP/2, follow redirects, client cert
curlie --http2 -L --cert client.pem --key client.key https://api.example.com/

# Pipe to jq — output auto-switches to raw bytes when not a TTY
curlie httpbin.org/get | jq '.headers'
```
