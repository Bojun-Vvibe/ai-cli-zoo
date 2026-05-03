# nebula

- **Repo:** https://github.com/slackhq/nebula
- **Version:** 1.10.3 (latest stable, 2026-02-06)
- **License:** MIT ([LICENSE](https://github.com/slackhq/nebula/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install nebula` · `pacman -S nebula` · static binary tarball from [GitHub releases](https://github.com/slackhq/nebula/releases) (Linux, macOS, Windows, FreeBSD, iOS, Android) · official Docker image at `nebulaoss/nebula` · the related `nebula-cert` binary ships in the same archive

## What it does

`nebula` is a peer-to-peer **overlay mesh network** built on the Noise Protocol Framework (the same crypto family that powers WireGuard) plus a tiny PKI of self-signed CA certs. Every host runs the `nebula` binary, which opens a single UDP socket, exchanges Noise IK handshakes with peers, and exposes a virtual `nebula1` (Linux) / `utun` (macOS) interface with a fixed private IP from a CIDR you control. Once two hosts have handshaken, traffic flows **directly** between them over UDP — no central relay sits in the data path. The control plane is a small fleet of `lighthouse` nodes (in practice 2–4 hosts on stable public IPs) whose only job is to answer "what is host X's current public `ip:port`?" so that two NATed peers can hole-punch and start talking. Identity is a `host.crt` issued offline by `nebula-cert` against a `ca.crt` you generated once; each cert binds a hostname, a virtual IP, and a list of **groups** (`db`, `web`, `prod`, `laptop`, …) that the firewall config keys off. The on-host firewall config (`config.yaml`) is the policy surface — rules look like `allow tcp 5432 from group:web to group:db` and are evaluated locally by every node, so a compromised laptop cert cannot reach `group:db` even if the network would otherwise route the packet. The threat model assumes the lighthouse is *not* trusted with traffic content (it only knows endpoint metadata) and that the CA private key lives offline on a hardened host. v1.7 added the **relay** feature for double-NAT cases where direct hole-punching fails (traffic transits a chosen peer, still encrypted end-to-end). v1.9 added **certificate v2** with shorter, faster Curve25519 signatures and tightened the cert lifetime story. v1.10.x is the current line, with focus on iOS/Android client polish, Windows TUN performance, and `nebula-cert` ergonomics. The whole stack is one Go binary plus YAML — no kernel module, no userspace dependencies, no external database.

## When to pick it / when not to

Pick `nebula` when (a) you have **dozens to thousands of hosts across many networks** (offices, home laptops, cloud VPCs, customer sites, IoT devices) that need to address each other by stable internal IP without depending on each cloud's VPC peering or each office's site-to-site VPN; (b) you want a **per-host firewall enforced by identity, not by network position** — a cert in `group:laptop` cannot reach `group:db` whether the laptop is in the office or on hotel Wi-Fi; (c) you are uncomfortable putting all your trust into a SaaS coordinator and want a fully self-hosted control plane (the lighthouses are *your* hosts); (d) you need predictable, low-latency direct paths between peers because your traffic is bursty/large (file sync, database replication, video) and you don't want a relay tax on every packet. Slack built and runs Nebula at production scale (the original 2019 blog post described tens of thousands of nodes), and the deploy shape — flat /16 overlay, group-based firewall, lighthouse pair per region — has been widely copied. Pair with [`step-cli`](../step-cli/) when you want a more flexible CA than `nebula-cert` for issuing host certs from CI; pair with [`consul`](../consul/) or [`tailscale`](../tailscale/) MagicDNS for service discovery on top of the overlay (Nebula itself does not provide DNS); pair with [`syncthing`](../syncthing/) or [`mutagen`](../mutagen/) for file sync over the overlay where ZeroTrust posture matters.

Skip Nebula when (a) you have **<10 hosts and no compliance pressure** — the cert-management overhead is not worth it, and a managed mesh like Tailscale is dramatically faster to stand up; (b) you need **zero-config, SSO-driven onboarding** for a workforce — Tailscale/Headscale's identity-provider integration is far slicker than handing every laptop a freshly minted Nebula cert; (c) you need **MagicDNS / short-name resolution** out of the box — Nebula intentionally stays out of the DNS layer; (d) you need **Layer-2 semantics** (broadcast, multicast, mDNS over the overlay) — Nebula is L3-only by design; (e) the deployment target is a hostile-MDM corporate laptop where you cannot ship a TUN-creating binary that needs CAP_NET_ADMIN.

## Vs already cataloged

- **Vs [`tailscale`](../tailscale/) / [`headscale`](../headscale/):** the closest comparison. Tailscale is a managed WireGuard mesh with a SaaS coordinator (`controlplane.tailscale.com`); headscale is the self-hosted FOSS reimplementation of that coordinator. Tailscale wins on UX, identity-provider SSO, MagicDNS, ACL editor, and "install the app and click sign in". Nebula wins on (a) fully self-hosted control plane out of the box, (b) per-host firewall as the *primary* policy surface, (c) MIT-licensed end to end (the Tailscale client is BSD but the coordinator is closed; headscale closes that gap), and (d) explicit group-based identity instead of tag-based ACLs evaluated by a coordinator. For a small team, pick Tailscale. For a fleet you fully self-host, the choice is a real tradeoff.
- **Vs [`cloudflared`](../cloudflared/):** different problem. `cloudflared` exposes specific local services through Cloudflare's edge as a tunnel; it is not a peer-to-peer mesh. Use cloudflared when you want one HTTPS service reachable from the internet without opening firewall ports; use Nebula when you want N hosts to address each other directly.
- **Vs [`frp`](../frp/) / [`bore`](../bore/) / [`sshuttle`](../sshuttle/) / [`wstunnel`](../wstunnel/):** all are tunnel/relay tools for forwarding specific ports through a relay. Nebula is mesh, not tunneling — every node is both client and server.
- **Vs [`mosh`](../mosh/) / [`upterm`](../upterm/) / [`sshx`](../sshx/) / [`tmate`](../tmate/):** orthogonal — those are remote-shell tools that run *on top of* whatever network you have. Nebula is the network layer underneath.
- **Vs [`wireguard-tools`](https://www.wireguard.com/) (not in catalog):** Nebula uses Noise (closely related to WireGuard's crypto) but adds the lighthouse discovery layer, cert-based identity, and host-side firewall. WireGuard alone gives you point-to-point tunnels with manual key management; Nebula gives you a mesh with PKI and policy.
- **Vs [`tailspin`](../tailspin/) / [`tspin`](../tspin/):** unrelated — those are log-highlighting tools that share a name prefix with tailscale by coincidence.

## Caveats

- **The CA private key is the entire trust root.** If `ca.key` leaks, an attacker can mint a host cert for any IP in any group and join your overlay. Generate the CA on an offline host, store the key in a hardware token or sealed [`vault`](../vault/) blob, and rotate by issuing a new CA + cross-signing rather than ad-hoc.
- **Cert lifetime is your renewal cadence.** `nebula-cert sign -duration 8760h` (1 year) is common but means a stolen laptop's cert is valid for up to a year unless you maintain a CRL or rotate the CA. Shorter lifetimes (30–90d) plus automated renewal from a CI pipeline is the modern shape.
- **NAT traversal works but is not magic.** Symmetric NAT on both ends will fall back to relay (since v1.7) or fail; CGNAT mobile connections are the worst case. Plan for at least one relay-capable peer per region.
- **Lighthouses are an HA concern.** A single lighthouse means a single point of discovery failure for new sessions (existing sessions keep working). Run ≥2 lighthouses per region with stable public IPs and resilient hosting.
- **No MagicDNS, no service discovery.** You get stable internal IPs, not names. Layer your own `/etc/hosts` (small fleets), Consul/CoreDNS (medium), or an internal DNS zone served *over* the overlay (large) on top.
- **MTU pinning matters.** The default 1300 MTU is conservative; tuning is per-link. Misconfigured MTU is the most common "TCP works but large transfers stall" symptom.
- **iOS/Android clients exist but require network-extension entitlements.** Distribution outside TestFlight / Play Store internal testing is non-trivial — most fleets restrict mobile to a subset of users with the official builds.
- **Firewall is *deny by default* once you write any rule.** A common foot-gun is enabling outbound rules and forgetting inbound — the host will appear "down" from peers because ICMP/SSH inbound is implicitly blocked. Use `allow icmp from any to any` during bring-up, then tighten.
- MIT ([LICENSE](https://github.com/slackhq/nebula/blob/master/LICENSE)) — permissive; safe to bundle into commercial products and to ship as part of customer-side appliances.

## Example invocations

```bash
# Install
brew install nebula                         # macOS
sudo pacman -S nebula                       # Arch
# or grab the static binary tarball from GitHub releases

# 1. One-time: create a CA on an offline / hardened host
nebula-cert ca -name "Acme Corp"            # produces ca.crt + ca.key

# 2. Issue a host cert (do this per node, on the CA host, then ship cert+key to the node)
nebula-cert sign -name "lighthouse1" -ip "10.42.0.1/16" -groups "lighthouse"
nebula-cert sign -name "web-1"       -ip "10.42.1.1/16" -groups "web,prod"
nebula-cert sign -name "db-1"        -ip "10.42.2.1/16" -groups "db,prod"
nebula-cert sign -name "laptop-alice" -ip "10.42.10.5/16" -groups "laptop"

# 3. Print a cert (audit / debugging)
nebula-cert print -path web-1.crt

# 4. Run a node
sudo nebula -config /etc/nebula/config.yaml

# 5. Minimal config.yaml (excerpt) — host firewall is policy
# pki:
#   ca:   /etc/nebula/ca.crt
#   cert: /etc/nebula/host.crt
#   key:  /etc/nebula/host.key
# static_host_map:
#   "10.42.0.1": ["lh1.example.com:4242"]
# lighthouse:
#   am_lighthouse: false
#   hosts: ["10.42.0.1"]
# listen:
#   host: 0.0.0.0
#   port: 4242
# firewall:
#   outbound: [{port: any, proto: any, host: any}]
#   inbound:
#     - {port: 22,    proto: tcp, group: laptop}
#     - {port: 5432,  proto: tcp, group: web}
#     - {port: any,   proto: icmp, host: any}

# 6. Verify connectivity from one node to another
ping 10.42.2.1                               # db-1 from web-1
nebula -config /etc/nebula/config.yaml -test # validate config without starting the tunnel

# 7. Rotate a host cert (re-sign and reload)
nebula-cert sign -name "web-1" -ip "10.42.1.1/16" -groups "web,prod" -duration 720h
sudo systemctl reload nebula                 # SIGHUP picks up new cert
```
