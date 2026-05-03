# chisel

> **TCP / UDP tunnel over HTTP — `ssh -L` that traverses corporate
> proxies, CDNs, and NATs because it speaks plain HTTPS.**
> A single Go binary that runs in two modes: `chisel server` listens
> on an HTTP / HTTPS port and accepts encrypted SSH-over-WebSocket
> sessions; `chisel client` connects out from anywhere with HTTPS
> egress and forwards arbitrary local / remote ports through that
> session. Pinned to **v1.11.5**
> ([LICENSE](https://github.com/jpillora/chisel/blob/master/LICENSE),
> MIT).

Source: <https://github.com/jpillora/chisel>

## TL;DR

`chisel` exists because raw `ssh -R` does not survive the modern
network: corporate egress firewalls block port 22, hotel WiFi
captive portals only allow 80 / 443, CDNs in front of your
infrastructure want HTTP semantics, and Cloudflare / load balancers
in the path expect WebSocket frames not raw TCP. `chisel` wraps an
SSH-style encrypted session inside a WebSocket so the wire is
indistinguishable from any other HTTPS request, then exposes the
same `-L localport:remotehost:remoteport` and `-R remoteport:lhost
:lport` forwarding flags every shell user already knows. The whole
thing is one ~10 MB static Go binary; no daemon, no kernel module,
no Python, drop on the box and run.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install chisel

# Go install (any platform with a Go toolchain)
go install github.com/jpillora/chisel@latest

# One-line installer (downloads the right release tarball)
curl https://i.jpillora.com/chisel! | bash

# Docker
docker run --rm -it jpillora/chisel --help

# verify
chisel --version    # 1.11.5
```

Same binary is server *and* client — `chisel server` opens the
listener, `chisel client` connects out.

## License

MIT — see [LICENSE](https://github.com/jpillora/chisel/blob/master/LICENSE).
Permissive, no copyleft, safe to bundle into appliance images and
commercial products.

## One Concrete Example

```bash
# === Scenario: expose a service running on a laptop behind a NAT
# to a public-facing VM, the way `ngrok` would, but self-hosted ===

# 1. On the public VM (1.2.3.4), run the server.
chisel server \
  --port    8443 \
  --tls-key /etc/letsencrypt/live/tunnel.example/privkey.pem \
  --tls-cert /etc/letsencrypt/live/tunnel.example/fullchain.pem \
  --reverse \
  --auth    'shareuser:s3cret'

# 2. On the laptop, expose local :3000 (a dev server) as :8080 on
#    the public VM.
chisel client \
  --auth shareuser:s3cret \
  https://tunnel.example:8443 \
  R:8080:127.0.0.1:3000

# Anyone hitting http://tunnel.example:8080 now reaches the dev
# server on the laptop. Tunnel survives WiFi changes (auto-reconnect
# with exponential backoff); session is end-to-end encrypted with a
# fresh SSH key per session.

# === Scenario: SOCKS5 proxy through a single open HTTPS port ===
# 3. On the laptop, ask the server for a local SOCKS5 listener that
#    egresses through the server's network.
chisel client https://tunnel.example:8443 socks
# → SOCKS5 listening on 127.0.0.1:1080 — point a browser /
#   `curl --socks5 127.0.0.1:1080` / Slack at it; all traffic exits
#   from the public VM, so you reach intranet services as if you
#   were on that VM.

# === Scenario: jump-box style port forward from corp laptop ===
# 4. Forward the corp DB (db.intranet:5432, only reachable from the
#    bastion) to the laptop's localhost:5432, through chisel running
#    on the bastion.
chisel client https://bastion.example:8443 \
  5432:db.intranet:5432
psql -h 127.0.0.1 -p 5432 -U analyst warehouse
```

## Niche It Fills

**The "self-hosted ngrok / cloudflared" tunnel that runs anywhere
TCP / UDP can be routed through HTTPS.** Three real workloads it
absorbs cleanly:

- "Demo my laptop dev server to a customer" without paying for
  ngrok / Cloudflare Tunnel and without exposing the laptop's IP.
- "Reach a private service from a coffee shop" without standing up
  a full WireGuard / Tailscale mesh — one HTTPS port on a $5 VPS
  is the entire infrastructure.
- "Port-forward through a load balancer" — deploy `chisel server`
  behind your existing nginx / ALB / Cloudflare with HTTPS
  termination and the WebSocket upgrade just works.

## Why use it

Three things `chisel` does that the obvious alternatives do not,
that pay back the introduction:

1. **HTTPS-shaped wire format passes through everything.** Plain
   `ssh -R` requires port 22 outbound, which corporate / hotel /
   stadium WiFi blocks half the time. `chisel`'s WebSocket
   transport is indistinguishable from any other HTTPS request and
   passes captive portals, deep-packet inspection, and CDN proxies
   that strip non-HTTP traffic.
2. **One binary, no daemon, no auth server.** Ship it onto a
   $5 VPS as `wget && chmod +x && ./chisel server`, point the
   client at it, done. Compare to standing up Cloudflare Tunnel
   (requires Cloudflare account + dashboard config) or Tailscale
   (requires control plane account, ACL editing, magic DNS) or a
   WireGuard mesh (requires per-peer key distribution + persistent
   keepalive + UDP punching).
3. **Per-user / per-route ACLs are flat config.** `--authfile
   users.json` accepts a JSON dict of `user:pass` → allowed
   forwarding rules, so giving a contractor read-only forward
   access to one staging port is a one-line edit, not an SSO
   integration.

For an LLM-CLI / agent workflow that ends up needing "expose this
local agent server to a webhook source that needs a public URL",
`chisel` is the boring self-hosted answer that does not require a
SaaS account or a credit card and can be torn down with `kill`.

## Vs Already Cataloged

- **Vs [`cloudflared`](../cloudflared/):** different posture.
  `cloudflared` is the right answer when you are already on
  Cloudflare and want their edge to terminate the tunnel
  (WAF, DDoS, geo-routing, SSO via Access). `chisel` is the
  right answer when the relay is *yours* and the only requirement
  is "egress through HTTPS to a box I control".
- **Vs [`tailscale`](../tailscale/):** different shape.
  `tailscale` is a mesh VPN — every node sees every other node by
  hostname, ACLs are global. `chisel` is point-to-point port
  forwarding — surgical exposure of a specific port to a specific
  relay, no mesh state to maintain.
- **Vs `ssh -R` / `autossh` (not cataloged):** `chisel` is the
  HTTPS-friendly replacement for the same workflow. Use plain
  `ssh -R` when port 22 outbound is open and the path does not
  involve a CDN; use `chisel` when either of those is false.
- **Vs `ngrok` / hosted tunnel SaaS (not cataloged):**
  `chisel` is the BYO-relay version. You give up the magic
  `*.ngrok-free.app` URL and the inspection UI; you keep the data
  plane and the egress IP.

## Caveats

- **Bring your own TLS.** `chisel server` will speak plain HTTP if
  you let it, which exposes the SSH session inside an unencrypted
  WebSocket — fine inside a private network, never on the public
  internet. Always run with `--tls-cert` / `--tls-key` (or behind
  a reverse proxy that terminates TLS).
- **Bring your own auth.** Default is no auth. `--auth user:pass`
  or `--authfile` is mandatory for any internet-facing deploy;
  treat anyone with the credentials as having shell-equivalent
  access to whatever you forward.
- **Reverse-port mode is opt-in on the server.** The server
  refuses `R:` requests from clients unless started with
  `--reverse`. This is a feature (limits blast radius of a leaked
  client credential) but a common first-time confusion.
- **No HTTP-aware features.** `chisel` is L4 (TCP / UDP) only — it
  does not parse HTTP, cannot route by `Host:` header, cannot
  rewrite paths, cannot inspect requests. Pair with a real reverse
  proxy ([`caddy`](../caddy/), nginx, traefik) on the relay if you
  need any of that.
- **Throughput is limited by the WebSocket framing overhead and
  the relay's CPU.** A single `chisel` session over a 1 Gb link
  on a $5 VPS will saturate the VPS CPU before it saturates the
  link. For bulk transfer, prefer a real VPN ([`tailscale`](../tailscale/),
  [`headscale`](../headscale/), WireGuard).
