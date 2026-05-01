# mitmproxy

> **Interactive HTTPS-intercepting proxy with a Python scripting
> hook** — sits as a man-in-the-middle on HTTP/1, HTTP/2, HTTP/3,
> WebSocket, raw TCP/UDP and TLS-wrapped variants of all of them,
> generates per-host certificates on the fly from a local CA
> (`~/.mitmproxy/mitmproxy-ca-cert.pem`), and exposes the
> intercepted flows through three front-ends — a console TUI
> (`mitmproxy`), a web UI (`mitmweb`), and a non-interactive
> recorder (`mitmdump`) — plus an addon API where you write
> Python event handlers (`def request(flow): ...` /
> `def response(flow): ...`) to inspect, mutate, replay, or
> block traffic mid-flight. Pinned to **v12.2.2** (commit
> `ab470e5397d8cb73b3c44b9ee75834e6cda83ae7`,
> [LICENSE](https://github.com/mitmproxy/mitmproxy/blob/main/LICENSE),
> MIT).

Source: <https://github.com/mitmproxy/mitmproxy>

## TL;DR

`mitmproxy` is the canonical "see and reshape what my app actually
sends over the wire" tool. Run `mitmproxy -p 8080` (or
`mitmweb --web-host 127.0.0.1` for a browser UI), point a client
at it (system proxy, `HTTPS_PROXY=http://127.0.0.1:8080`,
explicit SDK config, or transparent mode with
`pf` / `iptables`), trust the local CA cert once
(`mitmproxy-ca-cert.pem` into the OS / browser / language trust
store), and every TLS handshake to every host is silently
re-terminated by mitmproxy. Each flow is a row in the TUI you can
inspect (request line, headers, body, decoded JSON / protobuf
with the right addon, response, timing); `e` opens the request
in `$EDITOR` for surgical mutation before replay; `r` replays;
`w` writes the flow file; `R` reads one back. The Python
addon API turns the same flows into "before this leaves my
machine, rewrite the `Authorization` header / drop this field
/ block this domain / log this latency / mock this response."
`mitmdump` is the same engine without the UI, suited for CI
recordings and long-running captures.

## Install

```bash
# pipx (recommended — single-tool venv)
pipx install mitmproxy

# Homebrew
brew install mitmproxy

# binary release (single static-ish bundle)
curl -L https://downloads.mitmproxy.org/12.2.2/mitmproxy-12.2.2-macos-arm64.tar.gz \
  | tar -xz -C /usr/local/bin

# verify
mitmproxy --version       # mitmproxy 12.2.2

# trust the generated CA on macOS (one time, after first run)
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain ~/.mitmproxy/mitmproxy-ca-cert.pem
```

The CA cert is generated on first launch into `~/.mitmproxy/`;
trust it in the *client* (browser / OS / language runtime) you
want to intercept, not in mitmproxy itself.

## One Concrete Example

```bash
# 1. start mitmproxy on port 8080 with a small Python addon
cat > rewrite.py <<'EOF'
from mitmproxy import http

def request(flow: http.HTTPFlow) -> None:
    # strip a noisy header before it hits the upstream
    flow.request.headers.pop("X-Telemetry-Id", None)
    # rewrite a flag for local QA
    if flow.request.pretty_url.startswith("https://api.example.com/v1/feature"):
        flow.request.query["enable_beta"] = "1"

def response(flow: http.HTTPFlow) -> None:
    # mock a 500 to test client retry behaviour, but only once
    if flow.request.pretty_url.endswith("/checkout") and not getattr(response, "_fired", False):
        flow.response = http.Response.make(500, b"upstream sad", {"Content-Type": "text/plain"})
        response._fired = True
EOF

mitmproxy -s rewrite.py -p 8080

# 2. in another shell: point a client at it
HTTPS_PROXY=http://127.0.0.1:8080 \
  curl --cacert ~/.mitmproxy/mitmproxy-ca-cert.pem \
       https://api.example.com/v1/feature?x=1

# 3. CI use: record a session non-interactively, then replay later
mitmdump -w session.flows --set block_global=false
mitmdump -nr session.flows -s rewrite.py    # offline replay through the addon

# 4. browser-friendly UI on a remote box
mitmweb --web-host 0.0.0.0 --web-port 8081 -p 8080
```

## Niche It Fills

**Programmable HTTPS interception for engineers, not for spying.**
The client SDK you're integrating with says it sends X but the
server insists it received Y; an LLM agent wraps an HTTP API and
you need to know exactly what it sent on the wire; a third-party
mobile SDK is leaking PII and you have one afternoon to find out
which field. mitmproxy is the canonical tool for all three —
and its Python addon API means the answer to "now mutate this
in flight" is 5 lines, not a fork of nginx.

## Vs Already Cataloged

- **Vs [`curl`](../curl/) / [`xh`](../xh/) / [`httpie`](../httpie/):**
  those are *clients you control end-to-end*. mitmproxy intercepts
  traffic from a client *you don't necessarily control* (a mobile
  app, a closed-source SDK, a JS bundle in a browser). Different
  layer.
- **Vs [`grpcurl`](../grpcurl/) / [`grpcui`](../grpcui/):** gRPC
  ad-hoc clients — you craft the request. mitmproxy can also
  decode gRPC (with the gRPC contentview), but the use-case is
  inspecting / mutating *real traffic from another client*, not
  authoring requests.
- **Vs [`bombardier`](../bombardier/) / [`oha`](../oha/) /
  [`plow`](../plow/):** load generators. Different layer entirely.
- **Vs [`tcpdump`](https://www.tcpdump.org/) / `tshark`:** packet
  capture, but TLS payloads are opaque. mitmproxy *terminates*
  TLS so you see the cleartext request / response.
- **Vs [`bruno`](../bruno/) / Postman:** GUI-first request
  composers / collection runners; opposite direction (author and
  send) from intercept-and-mutate.

## Caveats

- The CA-cert trust step is **mandatory and security-relevant.**
  Trusting `mitmproxy-ca-cert.pem` in your OS / browser means
  *anything that holds the matching private key* in
  `~/.mitmproxy/` can MITM your TLS to any host. Restrict the
  file (`chmod 600`), keep it on a personal dev box, and **never
  install it on a shared / production / personal-banking
  machine.** Untrust when done (`security delete-certificate`
  on macOS; equivalents for OS / browser stores on Linux /
  Windows).
- HTTP/3 (QUIC) interception is more recent and requires the
  client to honour the proxy hint; some apps QUIC-only and skip
  the proxy — fall back to forcing HTTP/2.
- Some apps **certificate-pin** and reject the mitmproxy CA on
  purpose (Snapchat, banking apps, SDKs with pinning). The
  intended workaround is `--ssl-insecure` on the *upstream* side
  + Frida / objection on the *client* side to disable pinning;
  if the app is not yours, this often crosses a ToS line.
- The Python addon API runs **inside the proxy event loop** —
  blocking I/O in `request()` blocks all flows. Use the `async`
  variants (`async def request(flow):`) for anything network-
  bound.
- `mitmdump -w session.flows` files contain *every header and
  body* of the captured traffic, including secrets. Treat
  `.flows` files as you would `.env` — never commit, never
  share.
