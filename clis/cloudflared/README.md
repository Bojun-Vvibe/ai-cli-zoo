# cloudflared

> **A Cloudflare Tunnel client that exposes a localhost service to
> the public internet over an authenticated, outbound-only TLS
> connection** — a single Go binary that creates a persistent
> tunnel from your machine to Cloudflare's edge, eliminating the
> need to open inbound firewall ports or own a public IP. Pinned
> to **v2026.3.0**
> ([LICENSE](https://github.com/cloudflare/cloudflared/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/cloudflare/cloudflared>

## TL;DR

`cloudflared` is what you run when you have a service on
`127.0.0.1:8080` (a dev server, a Jupyter notebook, a webhook
receiver, an LLM endpoint) and you need it reachable at a stable
URL from anywhere on the internet, without poking holes in your
NAT, without a static IP, and without paying for a VPS. It opens
an outbound TCP/QUIC connection to Cloudflare's edge, registers a
hostname (either a free `*.trycloudflare.com` quick-tunnel or your
own domain in a Cloudflare zone), and reverse-proxies inbound
HTTPS requests through that long-lived connection back to your
local port. Authentication, TLS termination, and DDoS protection
are handled at the edge; your machine sees only the proxied
requests on a loopback socket. The tunnel survives ISP IP changes,
laptop sleep / resume, and Wi-Fi switches because the client
re-dials. Two operating modes: **quick tunnels** (`cloudflared
tunnel --url http://localhost:8080`, no signup, random subdomain,
process-lifetime only) and **named tunnels** (signed-up account,
your own subdomain, persists across restarts, runs as a service).

## Install

```bash
# Homebrew (macOS / Linux)
brew install cloudflared

# Pre-built binaries (any OS / arch)
# macOS arm64:
curl -Lo cloudflared "https://github.com/cloudflare/cloudflared/releases/download/2026.3.0/cloudflared-darwin-arm64.tgz"
tar xzf cloudflared
sudo install cloudflared /usr/local/bin/

# Linux x86_64 .deb:
curl -LO "https://github.com/cloudflare/cloudflared/releases/download/2026.3.0/cloudflared-linux-amd64.deb"
sudo dpkg -i cloudflared-linux-amd64.deb

# Linux x86_64 .rpm:
sudo rpm -ivh "https://github.com/cloudflare/cloudflared/releases/download/2026.3.0/cloudflared-linux-x86_64.rpm"

# Docker
docker run -d --name cf-tunnel cloudflare/cloudflared:2026.3.0 tunnel run

# Windows
winget install --id Cloudflare.cloudflared

# verify
cloudflared --version    # cloudflared version 2026.3.0 (...)
```

No account needed for quick tunnels. Named tunnels require a
free Cloudflare account and a domain in a Cloudflare zone;
`cloudflared tunnel login` opens a browser to authorize the
client, drops `~/.cloudflared/cert.pem`, and from then on
everything is CLI.

## License

Apache-2.0 — see
[LICENSE](https://github.com/cloudflare/cloudflared/blob/master/LICENSE).
Permissive, requires preservation of NOTICE and patent grant
clause; safe for commercial redistribution.

## One Concrete Example

```bash
# 1. quick tunnel — zero signup, ephemeral subdomain
cloudflared tunnel --url http://localhost:8080
# → https://random-words-1234.trycloudflare.com points at :8080
#   for as long as this process runs

# 2. named tunnel — persistent, your own hostname
cloudflared tunnel login                       # browser auth, one-off
cloudflared tunnel create my-laptop            # writes ~/.cloudflared/<UUID>.json
cloudflared tunnel route dns my-laptop dev.example.com
cat > ~/.cloudflared/config.yml <<'YAML'
tunnel: my-laptop
credentials-file: /Users/me/.cloudflared/<UUID>.json
ingress:
  - hostname: dev.example.com
    service: http://localhost:8080
  - hostname: api.example.com
    service: http://localhost:9090
  - service: http_status:404                   # required catch-all
YAML
cloudflared tunnel run my-laptop

# 3. install as a launchd / systemd service
sudo cloudflared service install                # systemd on Linux, launchd on macOS
# → tunnel survives reboot, logs to journalctl / Console.app

# 4. tunnel a non-HTTP TCP service (e.g. SSH, Postgres)
# server side:
cloudflared tunnel route dns my-laptop ssh.example.com
# config.yml:
#   - hostname: ssh.example.com
#     service: ssh://localhost:22
# client side:
cloudflared access ssh --hostname ssh.example.com
ssh -o ProxyCommand='cloudflared access ssh --hostname %h' me@ssh.example.com

# 5. expose a unix socket (e.g. Docker daemon)
# config.yml:
#   - hostname: docker.example.com
#     service: unix:/var/run/docker.sock

# 6. inspect tunnel status
cloudflared tunnel info my-laptop
# → connector IDs, edge POPs (LAX, SJC, …), uptime, traffic counters

# 7. rotate a tunnel's credentials without DNS change
cloudflared tunnel token --cred-file new.json my-laptop
```

## Niche It Fills

**The "I want a public URL for my localhost" problem, solved by
the host of a CDN rather than a third-party tunnel broker.**
Before `cloudflared`, the answers were `ngrok` (proprietary,
metered free tier, random subdomain), `localtunnel` (community,
flaky), or "rent a VPS and reverse-SSH to it" (operationally
heavy). `cloudflared` is the same idea — outbound-only TLS to a
broker that fronts a public URL — but the broker is Cloudflare's
existing edge network, the auth & DDoS-protection layer is the
same one fronting Cloudflare-hosted sites, and quick-tunnels are
unmetered and free. The tradeoff is account lock-in: the moment
you want a stable hostname you are inside Cloudflare's DNS and
Zero Trust dashboards.

## Why use it

Three things `cloudflared` does that the alternatives do not, that
explain why it has eaten most of the homelab / hobbyist tunnel
market:

1. **Outbound-only, no inbound firewall changes.** The tunnel is
   one persistent TLS/QUIC connection from your machine *out* to
   Cloudflare's edge. No port forwarding on the home router, no
   NAT traversal hacks, no IPv6 prerequisite, no static IP. Works
   identically on a laptop on a coffee-shop Wi-Fi, a Raspberry Pi
   behind CGNAT, and a corporate network that allows outbound 443.
2. **Edge-terminated TLS + DDoS protection bundled.** The public
   hostname gets a Cloudflare-managed certificate (free, auto-
   renewed); inbound traffic hits Cloudflare's WAF/rate-limit/
   bot-management layer before reaching your tunnel. You do not
   run `certbot`, you do not size for traffic spikes, you do not
   own the abuse-handling problem.
3. **Free quick tunnels with no account or rate limit.**
   `cloudflared tunnel --url http://localhost:8080` works from a
   fresh install with no signup; the only cost is that the
   subdomain is random and dies with the process. For
   "demo this branch to a coworker right now" this is the lowest-
   friction option in the entire category.

For an LLM-CLI workflow, `cloudflared tunnel --url
http://localhost:11434` exposes a local `ollama` / `llama.cpp` /
`mlx_lm.server` endpoint to a remote agent runner with a single
command and HTTPS by default — useful for "agent runs in CI but
the model lives on my Mac".

## Vs Already Cataloged

- **Vs [`tailscale`](../tailscale/):** orthogonal models. Tailscale
  builds a private mesh VPN — every member device gets a stable
  `100.x.y.z` IP, traffic is end-to-end encrypted between peers,
  and there is no public URL. `cloudflared` builds a *public*
  ingress — anyone on the internet can hit the hostname,
  authentication is enforced at the Cloudflare edge (or not at all
  for quick tunnels). Use Tailscale for "my devices talking to
  each other"; use `cloudflared` for "the public internet talking
  to one of my devices".
- **Vs [`bore`](../bore/):** `bore` is the minimal-scope peer —
  a 200-line Rust TCP-tunnel server you can self-host (`bore
  server` on a VPS, `bore local 8080 --to my.vps:7835` on the
  laptop). Pick `bore` if you specifically want to *own* the
  broker and avoid Cloudflare; pick `cloudflared` if you want
  edge TLS, DDoS protection, and a free hosted broker.
- **Vs [`sshuttle`](../sshuttle/):** different verb. `sshuttle`
  is a *VPN over SSH* — it routes your laptop's outbound traffic
  through a remote SSH server. `cloudflared` is the opposite —
  it routes *inbound* traffic from the internet to your laptop's
  loopback. They compose: `sshuttle` for "my laptop's egress
  pretends to be a server in datacenter X"; `cloudflared` for
  "my laptop's loopback is reachable as a public URL".
- **Vs `ngrok` (not cataloged):** the historical incumbent. Same
  outbound-tunnel + public-URL model. Ngrok's free tier rate-
  limits and rotates subdomains every 8 hours; `cloudflared`
  quick tunnels are unmetered. Ngrok's paid tier offers features
  `cloudflared` doesn't (request inspector UI, replay,
  TCP-without-domain). Most homelab users moved to `cloudflared`
  in 2022–2023 when the free tier became unviable.

## Caveats

- **Vendor lock-in for named tunnels.** Persistent hostnames
  require the domain to be in a Cloudflare zone — moving the
  tunnel off Cloudflare means re-doing DNS, certificates, and any
  Zero Trust access policies you stacked on top. Quick tunnels
  are vendor-free in the sense that they are also throwaway.
- **Inbound traffic always passes through Cloudflare's edge.**
  This is the point — but it also means Cloudflare can see (and
  in principle log / inspect) the plaintext of every request,
  because TLS is terminated at the edge and re-encrypted (or
  not) on the leg back to your machine. For privacy-sensitive
  workloads consider end-to-end TLS (`origin-cert`) so the edge
  cannot read the plaintext.
- **Quick tunnels are explicitly not for production.** Subdomain
  rotates per process invocation, no SLA, no rate limit guarantee,
  and the `*.trycloudflare.com` zone is sometimes blocked by
  corporate proxies / school networks for being a known
  tunnelling category. Use named tunnels with your own domain
  for anything that needs a stable URL.
- **Tunnel restart drops in-flight connections.** Long-poll /
  websocket connections terminate cleanly but reconnects are the
  client's problem; HTTP/2 streams reset. The official
  recommendation is to run two `cloudflared` processes against
  the same named tunnel for HA — they share the load and
  one can restart while the other carries traffic.
- **Not a generic SOCKS proxy.** Each ingress rule maps one
  hostname to one upstream service URL. To expose 50 internal
  services you need 50 hostnames (or one hostname + path
  routing, with the limitation that `cloudflared` does not do
  L7 path rewriting — that has to live in the upstream).
- **Egress connection is sticky to one POP.** If the chosen
  Cloudflare data center has a partial outage, the connector
  may take 30–60 s to fail over. Use `cloudflared tunnel info
  --output json` to monitor connector locations in production.
