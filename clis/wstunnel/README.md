# wstunnel

> **Tunnel any TCP / UDP / Unix-socket traffic over plain HTTP,
> WebSocket, or HTTP/2** — so you can punch through corporate
> proxies, firewall captive portals, and stuck network paths that
> only allow port 80 / 443. Single Rust binary, server + client
> in one. Pinned to **v10.5.2** (SPDX: `BSD-3-Clause`,
> [LICENSE](https://github.com/erebe/wstunnel/blob/main/LICENSE)).

Source: <https://github.com/erebe/wstunnel>

## TL;DR

`wstunnel` is what you reach for when `ssh -L` is blocked but
HTTPS to the open internet still works. Run a `wstunnel server`
on a VPS you control on `:443`; from the locked-down laptop run
`wstunnel client wss://your-vps:443 -L tcp://2222:internal-host:22`
and you have a TCP forward that looks like ordinary WebSocket
traffic to the proxy. Supports TCP, UDP, Unix sockets,
SOCKS5/HTTP-CONNECT proxy modes, mTLS, and TLS-fronted reverse
tunnels initiated from the server side.

## Install

```bash
# Homebrew (macOS / Linux)
brew install wstunnel

# Cargo
cargo install --locked wstunnel

# Pre-built release binary (Linux x86_64)
curl -Lo wstunnel.tar.gz "https://github.com/erebe/wstunnel/releases/download/v10.5.2/wstunnel_10.5.2_linux_amd64.tar.gz"
tar xf wstunnel.tar.gz
sudo install wstunnel /usr/local/bin/

# Docker
docker pull ghcr.io/erebe/wstunnel:v10.5.2

# verify
wstunnel --version    # wstunnel 10.5.2
```

## License

BSD-3-Clause — see
[LICENSE](https://github.com/erebe/wstunnel/blob/main/LICENSE).

## One Concrete Example

```bash
# === server side (a VPS reachable on :443) ===
wstunnel server wss://0.0.0.0:443 \
  --restrict-to internal-host:22 \
  --restrict-to internal-host:5432

# === client side (the locked-down machine) ===

# 1. forward local TCP :2222 to internal-host:22 over wss
wstunnel client wss://your-vps:443 \
  -L tcp://2222:internal-host:22
ssh -p 2222 user@127.0.0.1

# 2. forward local TCP :5432 to a database behind the tunnel
wstunnel client wss://your-vps:443 \
  -L tcp://5432:internal-host:5432

# 3. UDP tunnel (e.g. WireGuard handshake)
wstunnel client wss://your-vps:443 \
  -L udp://51820:wg-host:51820

# 4. expose a local service to the server side (reverse tunnel)
wstunnel client wss://your-vps:443 \
  -R tcp://0.0.0.0:8080:127.0.0.1:3000

# 5. SOCKS5 over wss (browser / curl can speak SOCKS)
wstunnel client wss://your-vps:443 -L socks5://1080
curl --socks5 127.0.0.1:1080 https://example.internal/

# 6. HTTP CONNECT proxy mode
wstunnel client wss://your-vps:443 -L stdio://internal-host:22

# 7. tunnel through an upstream HTTP proxy that demands auth
wstunnel client wss://your-vps:443 \
  --http-proxy http://user:pass@corp-proxy:8080 \
  -L tcp://2222:internal-host:22

# 8. mTLS client cert (pinned to a CA you control)
wstunnel client wss://your-vps:443 \
  --tls-certificate ~/.wstunnel/client.crt \
  --tls-private-key ~/.wstunnel/client.key \
  -L tcp://2222:internal-host:22
```

## Niche It Fills

**Tunnelling when the only thing the network lets out is HTTPS.**
A normal `ssh -D` SOCKS proxy gets blocked the moment the firewall
deep-packet-inspects port 22 traffic; `wstunnel` makes the same
forward look indistinguishable from a regular `wss://` WebSocket
upload. Single static binary, both ends, supports TCP / UDP /
Unix / SOCKS / HTTP-CONNECT / reverse forwards.

## When to use

1. **The egress path only allows :80 / :443 + TLS.** Hotel
   wifi, conference networks, restrictive corporate egress, some
   airline wifi.
2. **You need UDP across a TCP-only path.** WireGuard,
   QUIC, DNS, game protocols all carry over `wstunnel`'s UDP
   mode tunnelled inside WebSocket frames.
3. **You want a reverse tunnel without configuring `sshd`.**
   `-R` exposes a local service to the server side without
   touching SSH config.
4. **You want one binary on both ends.** Same Rust binary acts
   as server (`wstunnel server`) or client (`wstunnel client`)
   based on the first arg.

## When NOT to use

- **The network permits direct outbound SSH.** Plain
  `ssh -L`/`ssh -D` is simpler, has no extra hop, and benefits
  from the existing `~/.ssh/config` ergonomics.
- **You need a full mesh / multi-peer VPN.** `wstunnel` is
  point-to-point. Use WireGuard or Tailscale for mesh topologies.
- **You need a managed, auditable corporate solution.**
  `wstunnel` is operator-grade — no audit trail, no SSO, no
  centralised policy. Use Cloudflare Tunnel / Tailscale / a real
  ZTNA product when those are requirements.
- **The remote server admin has not authorised this.** Bypassing
  network egress controls without permission is a policy
  violation in most workplaces. Self-host on your own VPS.

## Vs Already Cataloged

- **Vs [`croc`](../croc/):** orthogonal — `croc` is one-shot
  file transfer (peer-to-peer with relay fallback), `wstunnel`
  is persistent socket forwarding.
- **Vs [`websocat`](../websocat/):** related but different
  granularity. `websocat` is a *netcat for a single WebSocket
  connection* — great for poking at one endpoint, not for
  multi-flow port forwarding. `wstunnel` manages a long-lived
  tunnel that multiplexes many connections over one upgraded
  WebSocket.
- **Vs [`syncthing`](../syncthing/):** different problem —
  `syncthing` synchronises filesystems, `wstunnel` forwards live
  sockets.

## Caveats

- **You need a server you control on the open internet.** No
  managed relay. Cheapest setup is a $5/month VPS with `:443`
  exposed and a TLS cert (e.g. via Caddy or `--tls-certificate`).
- **Detectable by motivated DPI.** Looks like normal `wss://` to
  most filters but a network operator who actively fingerprints
  WebSocket sub-protocols can flag the traffic. Not designed as
  an anti-censorship tool — designed as a connectivity tool.
- **Latency overhead.** Two TLS handshakes (client→server, then
  server→target) add measurable RTT vs a direct path. Fine for
  SSH and DB clients; not a substitute for a direct circuit.
- **`v10.x` config syntax differs from `v6.x`.** Old blog posts
  show pre-v7 flags (`-L L:ws-host:port:tgt:port`). The modern
  syntax is URL-style (`-L tcp://localport:tgt:port`). Read the
  current `--help`, not Stack Overflow from 2020.
