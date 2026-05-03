# frp

> **Fast reverse proxy** that exposes a service behind a NAT or
> firewall to the public internet (or to another private
> network) by running a small client (`frpc`) on the inside that
> dials out to a server (`frps`) on a public host and tunnels
> TCP / UDP / HTTP / HTTPS / STCP / SUDP / XTCP traffic back
> through the established connection. The classic self-hosted
> alternative to ngrok / Cloudflare Tunnel: you own both the
> server and the client, you control the auth and the bind
> ports, and the protocol is open. Pinned to **v0.68.1**
> ([LICENSE](https://github.com/fatedier/frp/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/fatedier/frp>

## TL;DR

Rent a $5 VPS with a public IP, run `frps` on it. On your laptop
/ home server / Raspberry Pi behind NAT, run `frpc` pointing at
the VPS. Define one or more `proxies` in the client config — TCP
port forward, HTTP vhost, HTTPS SNI, P2P hole-punched STCP — and
the inside service is reachable from the public internet via the
VPS, with no router port-forward, no UPnP, and no third-party
relay holding your traffic.

## Install

```bash
# macOS
brew install frpc        # client
brew install frps        # server (less common on a laptop)

# Pre-built tarballs (server + client in one)
curl -LO https://github.com/fatedier/frp/releases/download/v0.68.1/frp_0.68.1_linux_amd64.tar.gz
tar xf frp_0.68.1_linux_amd64.tar.gz
cd frp_0.68.1_linux_amd64
sudo install frps frpc /usr/local/bin/

# Docker
docker pull snowdreamtech/frpc:0.68.1
docker pull snowdreamtech/frps:0.68.1

# Linux package managers
# Arch:  pacman -S frp
# Nix:   nix-env -iA nixpkgs.frp

# verify
frpc -v   # 0.68.1
frps -v   # 0.68.1
```

Config files default to TOML at `frpc.toml` / `frps.toml` in the
working dir; pass `-c /path/to/config.toml` to relocate. The 0.5x
INI format is still parsed but deprecated.

## License

Apache-2.0 — see
[LICENSE](https://github.com/fatedier/frp/blob/master/LICENSE).
Permissive with patent grant; suitable for commercial
self-hosting and forks.

## One Concrete Example

```bash
# === On the public VPS (frps.toml) ===
cat > frps.toml <<'EOF'
bindPort = 7000
auth.method = "token"
auth.token = "change-me-32-chars-of-entropy"

# Optional: HTTP / HTTPS vhost ports for "expose this web app"
vhostHTTPPort  = 80
vhostHTTPSPort = 443

# Optional: web dashboard at :7500 (basic auth)
webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "another-secret"
EOF
frps -c ./frps.toml

# === On the inside box (frpc.toml) ===
cat > frpc.toml <<'EOF'
serverAddr = "vps.example.com"
serverPort = 7000
auth.method = "token"
auth.token  = "change-me-32-chars-of-entropy"

# Expose local SSH on a fixed remote port
[[proxies]]
name       = "home-ssh"
type       = "tcp"
localIP    = "127.0.0.1"
localPort  = 22
remotePort = 2222

# Expose a local web app at a public hostname
[[proxies]]
name        = "blog"
type        = "http"
localIP     = "127.0.0.1"
localPort   = 8080
customDomains = ["blog.example.com"]

# P2P (no traffic via the VPS once the hole is punched)
[[proxies]]
name      = "p2p-rdp"
type      = "xtcp"
secretKey = "rdp-shared-secret"
localIP   = "127.0.0.1"
localPort = 3389
EOF
frpc -c ./frpc.toml

# Now: ssh -p 2222 user@vps.example.com  -> hits home box
#      curl http://blog.example.com      -> hits local :8080
```

## Niche It Fills

**Self-hosted, protocol-agnostic NAT punch-through with a
config-file contract.** Cloud-managed tunnels (ngrok, Cloudflare
Tunnel, Tailscale Funnel, localtunnel) are easier on day one but
put a third party on the data path, gate features behind a paid
plan, and rotate URLs on free tiers. `frp` is the "I have a $5
VPS and I want a permanent reverse tunnel that I own end to end"
answer — and the protocol matrix (TCP / UDP / HTTP / HTTPS /
STCP / SUDP / XTCP) covers the cases ngrok-style HTTP-only
tunnels do not (game servers, RDP, mosh, raw TCP databases).

## Why use it

1. **You own both ends.** Server is one binary on a VPS; client
   is one binary on the inside box; the wire protocol is open.
   No vendor can rate-limit you, change pricing, or take the
   service offline.
2. **Protocol coverage beyond HTTP.** `tcp` / `udp` for raw
   ports, `http` / `https` for vhost-routed web apps with
   per-route basic auth, `stcp` / `sudp` for "secret"-gated
   point-to-point that does not appear in the dashboard, `xtcp`
   for P2P hole-punching that takes the VPS off the data path
   once the connection is established.
3. **Stable URL / port.** Unlike free ngrok, the public bind
   (`remotePort`, `customDomains`) is whatever you put in the
   config — fine for DNS records, bookmarks, and `~/.ssh/config`
   entries.
4. **Cheap.** A single-core, 512 MB VPS handles dozens of
   tunnels with negligible CPU; the only cost-shaped resource
   is the VPS bandwidth bill.

## Vs Already Cataloged

- **Vs [`cloudflared`](../cloudflared/):** `cloudflared` is the
  managed-Cloudflare-Tunnel client — zero infra to run, free
  tier is generous, but Cloudflare terminates TLS and sees the
  decrypted traffic, and the egress is whatever Cloudflare's
  edge gives you. `frp` is the self-hosted "I want my own VPS
  on the data path" alternative; pick `cloudflared` for the
  fastest setup, `frp` for the most control.
- **Vs [`tailscale`](../tailscale/):** Tailscale is a mesh VPN
  (every node sees every other node, ACL-gated); `frp` is a
  reverse-tunnel proxy (one client exposes specific services
  through one server). Tailscale is the better answer for "give
  my devices private addresses to talk to each other"; `frp` is
  the better answer for "expose this one inside service to the
  public internet on a stable public port."
- **Vs [`ttyd`](../ttyd/) / [`caddy`](../caddy/) /
  [`traefik`](../traefik/):** orthogonal — those are the
  *application* (web terminal, web server, ingress) on the
  inside that you might *expose with* `frp`. A common stack is
  `caddy` listening on `127.0.0.1:8080` on the home box and
  `frp` exposing it as `https://app.example.com` via the VPS.

## Caveats

- **You operate two services.** The VPS-side `frps` and the
  inside-side `frpc` both need supervision (systemd / launchd /
  Docker restart policy), config under version control, and
  upgrade discipline — version skew between client and server
  occasionally breaks features.
- **Tokens are bearer secrets.** `auth.token` in `frpc.toml` /
  `frps.toml` grants full tunnel-creation rights; rotate when an
  inside box is decommissioned, and never check the configs
  into a public repo.
- **HTTP / HTTPS vhost mode terminates at `frps`.** TLS for
  `type = "https"` proxies is terminated on the VPS unless you
  configure pass-through (`type = "tcp"` on :443) or front
  `frps` with a TLS terminator you trust. Same shape as any
  reverse proxy — be deliberate about where the cert lives.
- **`xtcp` (P2P) needs symmetric-NAT escape.** Hole punching
  works for most home routers but fails behind carrier-grade
  NAT, some corporate firewalls, and a few mobile networks; the
  client transparently falls back to STCP-via-server when it
  cannot punch.
- **Open public bind ports = scan target.** A `tcp` proxy to a
  remote port on a public VPS shows up in Shodan within hours;
  put the actual auth (SSH keys, app login) on the inside
  service, not just on `frp`.
