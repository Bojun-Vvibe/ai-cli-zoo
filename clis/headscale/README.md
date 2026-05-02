# headscale

> **An open-source, self-hosted implementation of the Tailscale
> control plane — one Go binary (`headscale serve`) replaces the
> hosted `controlplane.tailscale.com` SaaS while leaving every
> other piece of the Tailscale ecosystem untouched: official
> Tailscale clients on macOS / iOS / Android / Windows / Linux
> connect to your `headscale` server instead, get assigned
> `100.x.y.z` addresses out of a CGNAT range, negotiate
> WireGuard tunnels peer-to-peer (control plane never sees
> packet payloads), and use DERP relays for NAT traversal —
> with full feature parity for ACLs (HuJSON policy file),
> tags + tag owners, MagicDNS, subnet routers, exit nodes,
> pre-auth keys (now bcrypt-hashed as of v0.28), OIDC SSO
> (Authelia / Keycloak / Google Workspace / etc.), and a
> companion `headscale` CLI for tailnet ops (`headscale users
> create`, `headscale nodes list`, `headscale preauthkeys
> create --reusable --tags tag:server`).** Pinned to **v0.28.0**
> (released 2026-02-04),
> [LICENSE](https://github.com/juanfont/headscale/blob/main/LICENSE),
> BSD-3-Clause.

Source: <https://github.com/juanfont/headscale>

## TL;DR

`headscale` is the **un-SaaS Tailscale**: same WireGuard data
plane, same official Tailscale clients (no fork required —
point them at your control server with `tailscale up
--login-server=https://headscale.example.com`), same MagicDNS
hostnames (`ssh laptop` works across the tailnet), same
ACL grammar (HuJSON policy file with `groups`, `tagOwners`,
`acls`, `ssh`, `autogroup:member` / `autogroup:tagged` /
`autogroup:self`), same pre-auth key / OIDC registration
flows — but with **no external dependency, no per-user pricing,
no upper bound on devices**, and the control plane runs on
your own VPS / homelab / k8s cluster against SQLite (default,
fine to ~1000 nodes) or PostgreSQL (recommended above that).
Storage is a single SQLite file plus a config YAML; backup is
`cp db.sqlite db.sqlite.backup`. Exposes a gRPC + REST API
(same shape as Tailscale's own admin API) so `headscale` CLI,
[third-party web UIs](https://github.com/juanfont/headscale#web-interface)
(headscale-admin, headplane), and your own automation all
speak one wire format. v0.28 added **tags-as-identity**
(tagged devices are first-class, not user-owned), bcrypt
hashing for pre-auth keys, and smaller partial map updates
that cut bandwidth on large tailnets.

## Install

```bash
# Debian / Ubuntu (official .deb)
curl -fsSL -o /tmp/headscale.deb \
  https://github.com/juanfont/headscale/releases/download/v0.28.0/headscale_0.28.0_linux_amd64.deb
sudo dpkg -i /tmp/headscale.deb

# Static binary (Linux / macOS / FreeBSD, amd64 / arm64)
curl -fsSL -o /usr/local/bin/headscale \
  https://github.com/juanfont/headscale/releases/download/v0.28.0/headscale_0.28.0_linux_amd64
chmod +x /usr/local/bin/headscale

# Docker (the most common deployment shape)
docker run -d --name headscale \
  -p 8080:8080 -p 9090:9090 \
  -v ./config:/etc/headscale -v ./data:/var/lib/headscale \
  headscale/headscale:0.28.0 serve

# Homebrew (macOS / Linux — community formula)
brew install headscale

# Verify
headscale version    # 0.28.0
```

## One Concrete Example

```bash
# 1. Generate a default config
headscale generate config > /etc/headscale/config.yaml
#    Edit server_url, listen_addr, ip_prefixes, and noise.private_key_path

# 2. Run the control server (under systemd in production)
headscale serve

# 3. Create a user (the namespace your devices register under)
headscale users create alice

# 4. Mint a reusable pre-auth key tied to a tag (devices register
#    as "tagged" — server identity, not user-owned)
headscale preauthkeys create --reusable --tags tag:web,tag:prod

# 5. On each Tailscale client, register against your control server
sudo tailscale up \
  --login-server=https://headscale.example.com \
  --authkey=hskey-auth-abcd-...                      \
  --advertise-tags=tag:web

# 6. Inspect the tailnet
headscale nodes list                  # shows IPs, hostnames, tags, last-seen
headscale nodes list --user alice
headscale routes list                 # subnet routes advertised by nodes
```

Minimal HuJSON ACL policy (`/etc/headscale/acl.hujson`):

```hujson
{
  "groups": {
    "group:admins": ["alice@", "bob@"],
  },
  "tagOwners": {
    "tag:web":    ["group:admins"],
    "tag:prod":   ["group:admins"],
  },
  "acls": [
    // admins reach everything
    { "action": "accept", "src": ["group:admins"], "dst": ["*:*"] },
    // web tier reaches prod tier on 5432 only
    { "action": "accept", "src": ["tag:web"], "dst": ["tag:prod:5432"] },
  ],
  "ssh": [
    // admins SSH into any tagged box as root via Tailscale SSH
    { "action": "accept",
      "src":    ["group:admins"],
      "dst":    ["autogroup:tagged"],
      "users":  ["root"] },
  ],
}
```

```bash
headscale policy set -f /etc/headscale/acl.hujson
headscale policy check                    # validates without applying
```

## License

[BSD-3-Clause](https://github.com/juanfont/headscale/blob/main/LICENSE),
SPDX `BSD-3-Clause`.

## Niche / positioning

Pick `headscale` over the hosted [`tailscale`](../tailscale/)
SaaS when **(a)** per-user pricing at fleet scale is the
problem, **(b)** the threat model demands no third-party sees
even the metadata of which devices exist, **(c)** an air-gapped
or sovereign-cloud deployment forbids reaching out to
`controlplane.tailscale.com`, or **(d)** the homelab aesthetic
of "I run my own thing" is the goal. The data plane is
unchanged — official Tailscale clients, same WireGuard, same
DERP fallback (point at a self-hosted DERP server too if you
want zero Tailscale-Inc. infra). Pick over raw
[`wireguard`](../wireguard/) when you want NAT traversal,
device enrolment, ACL grammar, OIDC, and MagicDNS without
hand-rolling them. Pick over [`netbird`](../netbird/) /
[`nebula`](../nebula/) / [`zerotier`](../zerotier/) only when
you specifically want the *Tailscale* client + protocol
ecosystem (those are alternative mesh-VPN designs with their
own clients). Pair with [`tailscale`](../tailscale/) (the
client side — `tailscale up` against your headscale server),
[`derper`](https://pkg.go.dev/tailscale.com/cmd/derper) (run
your own DERP relay), and any reverse proxy ([`caddy`](../caddy/)
/ [`traefik`](../traefik/)) for TLS termination of the
headscale HTTPS surface. Skip when fewer than ~5 devices —
the Tailscale free tier covers 100 devices / 3 users and the
hosted control plane has zero ops cost; headscale earns its
keep at organisational scale or on principle.
