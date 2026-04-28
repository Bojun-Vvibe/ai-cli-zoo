# hurl

- **Repo:** https://github.com/Orange-OpenSource/hurl
- **Version:** 8.0.0 (latest stable, April 2026)
- **License:** Apache-2.0 ([LICENSE](https://github.com/Orange-OpenSource/hurl/blob/master/LICENSE))
- **Language:** Rust (built on libcurl)
- **Install:** `brew install hurl` · `cargo install hurl` · `pacman -S hurl` · `apt install hurl` (Debian 13+) · `winget install hurl` · static binaries on the GitHub release page · binary names are `hurl` and `hurlfmt`

## What it does

`hurl` is a plain-text HTTP runner: you write requests in a `.hurl`
file using a syntax that reads almost exactly like the raw HTTP wire
format (`GET https://example.com/api/users`, header lines, JSON or
multipart body, query string), then `hurl` executes them through
**libcurl** so every redirect, TLS, HTTP/2, HTTP/3 (QUIC), proxy,
client-cert, NTLM, and Brotli quirk behaves identically to `curl`. The
upgrade over `curl` is everything that comes *after* the request:
**chained scenarios** (the next request can reference values captured
from the previous one), **typed assertions** on status, headers, body
shape, JSONPath / XPath / regex selections, durations, redirect counts,
TLS certificate properties, and byte-level checks (`md5`, `sha256`,
`bytes`); **captures** that pin extracted values to variables and feed
them into later requests; **per-environment variables** via
`--variable`, `--variables-file`, or `HURL_var=value` env injection;
**parallel execution** of independent files; and a **report bundle**
(`--report-html`, `--report-junit`, `--report-tap`,
`--report-json`) so the same `.hurl` you use to debug an endpoint
locally drops straight into CI as a contract / smoke / API-integration
test. The companion `hurlfmt` reformats `.hurl` files canonically and
converts to/from `curl`, JSON AST, or HTML.

## When to pick it / when not to

Pick `hurl` whenever the question is "does this HTTP endpoint behave
correctly" and you want the same artifact to serve interactive
debugging, smoke testing, integration testing in CI, and synthetic
production probes. It is the right tool for API contract tests in any
language (the test file is text, not your service's stack), for
multi-step OAuth / login / CSRF flows where step N needs a token from
step N-1, for verifying that a deploy is actually serving the right
build (assert `X-Build-SHA` header), and for replacing a sprawl of
hand-rolled `bash + curl + jq + grep` scripts with one readable file
that fails loudly with line numbers when an assertion breaks. Pair it
with [`xh`](../xh/) or [`httpie`](../httpie/) for one-shot interactive
exploration before you commit a request to a `.hurl` file, with
[`oha`](../oha/) when the question shifts from "does it work" to "does
it scale", and with [`jq`](../jq/) for one-off transforms outside the
test loop (inside a `.hurl` file you usually use JSONPath captures
instead).

Skip it for **load testing** — `hurl --repeat` and `--parallel` exist
but the tool is built for correctness, not throughput; reach for
[`oha`](../oha/), `wrk`, `k6`, or `vegeta`. Skip it for **stateful
browser flows** — anything that depends on JavaScript execution,
DOM events, or service-worker behaviour needs a real browser driver
(Playwright, [`browser-use`](../browser-use/)). Skip it for **gRPC /
GraphQL subscription / WebSocket / SSE** as a primary protocol — gRPC
isn't supported, GraphQL works fine over HTTP POST but you write the
request manually, WebSocket / SSE need `websocat` or a real client.
Skip it as a **mock server** — `hurl` is a client; pair with
`mockoon`, `wiremock`, or `prism` if you need the other side. Finally,
the per-request error messages are good but the *file-level* parse
errors can be terse — keep `hurlfmt --check` in your editor's
on-save hook so you catch syntax mistakes before runtime.

## Example invocations

```bash
# Run one .hurl file
hurl login.hurl

# Run many in parallel and write an HTML report
hurl --test --parallel --jobs 8 --report-html out/ tests/*.hurl

# Run with environment variables (multiple sources, last wins)
hurl --variable env=staging --variable token=$TOKEN api.hurl
hurl --variables-file vars.staging.env api.hurl

# Capture values from one request and reuse them
cat <<'EOF' > flow.hurl
POST https://api.example.com/login
{ "user": "{{user}}", "pass": "{{pass}}" }

HTTP 200
[Captures]
session_id: jsonpath "$.session.id"
csrf:       header "X-CSRF-Token"

GET https://api.example.com/me
X-CSRF-Token: {{csrf}}
Cookie: SID={{session_id}}

HTTP 200
[Asserts]
jsonpath "$.email" == "{{user}}@example.com"
duration < 500
EOF
hurl --variable user=alice --variable pass=$PW flow.hurl

# JUnit XML for CI integration
hurl --test --report-junit junit.xml tests/*.hurl

# Convert a curl one-liner into a .hurl file you can grow into a test
echo "curl -X POST https://api.example.com/v1/items -H 'X-Token: abc' -d '{\"k\":1}'" \
  | hurlfmt --in curl --out hurl

# Reformat all .hurl files in-place (CI-friendly check mode)
hurlfmt --in-place tests/*.hurl
hurlfmt --check tests/*.hurl

# Verbose libcurl trace for debugging a single failing request
hurl --very-verbose flow.hurl 2>&1 | less -R
```
