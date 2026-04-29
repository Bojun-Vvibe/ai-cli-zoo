# websocat

> **`netcat` and `curl` for WebSockets — a single Rust binary that
> connects, listens, proxies, and pipes raw bytes across the
> WebSocket protocol from your shell.** Pinned to **v1.14.1**
> ([LICENSE](https://github.com/vi/websocat/blob/master/LICENSE),
> MIT).

Source: <https://github.com/vi/websocat>

## TL;DR

`websocat` is what you reach for the moment you need to talk to a
WebSocket endpoint from a terminal — to debug a chat backend, smoke-
test a real-time pricing feed, replay a captured frame, or stand up
a quick local echo server while you build a browser client. It
mirrors `nc` (netcat) ergonomics: stdin goes out as WebSocket
messages, incoming messages come to stdout, and you compose
endpoints by URL scheme. Client mode is `websocat
wss://example.com/socket`. Server mode is `websocat -s 8080`.
Anything in between — TLS, custom headers, subprotocol negotiation,
ping/pong tuning, message framing (text vs binary), reverse
proxying, splicing two sockets together — is a flag. Authored by
Vitaly Shukela; ~1.6k LOC of focused Rust.

## Install

```bash
# Homebrew (macOS / Linux)
brew install websocat

# Cargo (any platform with Rust >=1.70)
cargo install --version 1.14.1 websocat

# Pre-built binary (releases page)
curl -sSL -o /usr/local/bin/websocat \
  https://github.com/vi/websocat/releases/download/v1.14.1/websocat.x86_64-unknown-linux-musl
chmod +x /usr/local/bin/websocat

# Verify
websocat --version    # websocat 1.14.1
```

## License

MIT — see
[LICENSE](https://github.com/vi/websocat/blob/master/LICENSE).
Permissive, redistributable, vendoring inside a closed-source
product is fine, no copyleft on the bytes you push through the
socket. The binary statically links OpenSSL on most release
artifacts; a `-nossl` variant is published for environments that
forbid that.

## One Concrete Example

```bash
# 1. interactive client to a public echo server (stdin -> ws, ws -> stdout)
websocat wss://echo.websocket.org
# type a line, hit enter, see it echoed back

# 2. one-shot send and read one frame (good for smoke tests)
echo '{"op":"ping","ts":1714000000}' | \
  websocat -n1 wss://api.example.com/realtime

# 3. send a binary frame (base64-decode and pipe in)
base64 -d <<< 'AAECAwQ=' | websocat --binary wss://api.example.com/proto

# 4. add bearer auth + custom subprotocol negotiation
websocat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Sec-WebSocket-Protocol: v2.json" \
  wss://api.example.com/feed

# 5. local echo server on port 8080 (great for browser-side dev)
websocat -s 8080
# now any browser at ws://localhost:8080 echoes whatever you send

# 6. broadcast server: every message a client sends is fanned out
#    to every other connected client
websocat -E ws-l:127.0.0.1:8080 broadcast:mirror:

# 7. splice two endpoints — useful as a quick reverse proxy or
#    to bridge a TCP service to a WebSocket clientside
websocat --binary tcp-l:127.0.0.1:1234 wss://upstream.example.com/raw

# 8. replay a captured session from a file (one frame per line)
websocat -n -B 65536 wss://api.example.com/replay < frames.ndjson

# 9. tune ping/pong to keep a flaky NAT path warm
websocat \
  --ping-interval 15 \
  --ping-timeout 5 \
  wss://api.example.com/long-lived
```

## Niche It Fills

**Be `nc`/`curl` for WebSockets so you can debug, script, and bridge
real-time endpoints from a shell instead of writing a one-off Node
or Python program every time.** Browsers ship a WebSocket client,
and every backend language has a library, but neither helps when
you are SSHed into a bastion at 2 AM and need to confirm that the
upstream feed is actually emitting frames. `websocat` is the
shell-native primitive: one binary, stdin/stdout, URL-scheme
composability. It also doubles as a tiny dev server for the
inverse case — building a browser client without a backend yet —
because `-s 8080` gives you a working WebSocket server in one
command.

## Why use it

Three things `websocat` does that the alternatives do not:

1. **URL-scheme composition for non-trivial topologies.**
   `websocat` accepts not just `ws://` / `wss://` but also
   `tcp:`, `tcp-l:`, `unix:`, `unix-l:`, `cmd:`, `exec:`,
   `mirror:`, `broadcast:`, `reuse:`, `autoreconnect:` and chains
   them in a left-side / right-side pair. So `websocat
   tcp-l:127.0.0.1:1234 wss://upstream/raw` becomes a TCP-to-WS
   reverse proxy in one line, and `websocat -s 8080
   cmd:'./handler.sh'` runs a shell script per WebSocket message
   without writing a server.
2. **Both sides of the protocol from one binary.** Most
   alternatives are client-only (browser DevTools, `wscat`) or
   library-only (you write a server). `websocat` does interactive
   client (`websocat wss://...`), one-shot client (`-n1`),
   listening server (`-s`), and bidirectional splice in the same
   tool. That means the same binary you used to debug the
   production feed can stand up the local mock for your browser
   tests.
3. **Reasonable defaults for ops use — pings, reconnects, frame
   sizes.** WebSocket connections through corporate proxies and
   load balancers die silently if you don't ping; `websocat` has
   `--ping-interval` / `--ping-timeout` /
   `autoreconnect:wss://...` as first-class flags. Browser
   DevTools won't help you here; library code requires you to
   reimplement the keepalive logic each time.

For an integration test that needs to assert "the realtime
endpoint emits at least one frame within 5 seconds of subscribe,"
the entire test is `timeout 5 websocat -n1 wss://api/feed | jq -e
.event` in a CI step — no Node, no Python, no SDK install.

## Vs Already Cataloged

- **Vs [`hurl`](../hurl/):** different protocol. `hurl` scripts
  HTTP request/response flows with assertions and captures; it
  does not speak WebSocket framing. Use `hurl` for the REST
  handshake that issues the auth token, then pipe the token into
  `websocat` for the realtime channel.
- **Vs [`grpcurl`](../grpcurl/):** sibling tools for adjacent
  protocols. `grpcurl` is HTTP/2 + gRPC framing with JSON in/out;
  `websocat` is HTTP/1.1 Upgrade + WebSocket framing with
  arbitrary bytes in/out. They cover the two main "long-lived
  bidirectional channel" technologies you'll find behind modern
  realtime backends.
- **Vs [`curlie`](../curlie/) / `curl`:** `curl` can technically
  do `--include` against `ws://` to inspect the Upgrade
  handshake, but it cannot then send frames. `websocat` is what
  takes over once the 101 Switching Protocols line is on the
  wire.
- **Vs [`httpie`](../httpie/):** same gap as `curl` — HTTPie has
  no first-class WebSocket frame support in core. `websocat` is
  the WebSocket-native counterpart in the same shell-friendly
  ergonomic family.

## Caveats

- **Frame framing is your problem if you mix text and binary.**
  By default `websocat` sends text frames; `--binary` switches
  the whole session to binary. If your protocol mixes both per-
  message, you need `multimessage:` overlays or a small wrapper
  script. Pure-text JSON protocols (most chat / pricing feeds)
  are fine on defaults.
- **Lots of features behind small flags — read `--long-help`.**
  The short `--help` is intentionally terse; the full surface
  (every URL-scheme overlay, every framing option, every
  reconnect / mirror / broadcast variant) is in `websocat
  --long-help` and the README's "Examples" section. The flag
  density is high because the design is "small composable
  primitives," not "one big config object."
- **TLS verification can be skipped — don't ship that to prod.**
  `--insecure` skips cert verification, which is fine for
  staging-with-self-signed but a foot-gun in production scripts.
  No flag will warn you; treat its presence in committed code as
  a review red flag.
- **No automatic JSON pretty-print on receive.** Frames come to
  stdout exactly as the server sent them. Pipe through `jq` for
  human-readable JSON. (`websocat ... | jq .` works, since each
  text frame is one line by default.)
- **Single-process, single-machine — not a load generator.** For
  WebSocket load testing reach for [`k6`](../k6/) with its
  WebSocket module, which can ramp thousands of concurrent
  virtual users with assertions and metrics.
