# tailscale

> **Zero-config WireGuard mesh VPN** — every device runs a single
> `tailscaled` daemon that authenticates against an identity
> provider (Google / GitHub / Okta / passkey / OIDC),
> registers in your "tailnet," and gets a stable
> `100.x.y.z` IPv4 + IPv6 ULA address; pairwise WireGuard tunnels
> are established on demand through NAT-traversal helpers (DERP
> relays as fallback) so any node can `ssh user@laptop` /
> `curl http://nas:8080` from anywhere on the internet, with
> optional layered features — MagicDNS (DNS for the tailnet),
> ACL-as-code (`HuJSON` policy file), Subnet routing, Exit nodes
> (route 0.0.0.0/0 through one tailnet member), Tailscale SSH
> (auth replaces SSH keys), Funnel (expose a tailnet HTTPS service
> to the public internet), Taildrop (file send between devices).
> Pinned to **v1.96.4** (commit
> `bdf3419e7d5c851b9fe4c974d93ffb6f0da4820d`,
> [LICENSE](https://github.com/tailscale/tailscale/blob/main/LICENSE),
> BSD-3-Clause).

Source: <https://github.com/tailscale/tailscale>

## TL;DR

`tailscale` removes the YAK-shave from "I want my laptop, my home
NAS, my cloud VM, and my phone all to talk to each other on a
private flat network" — you `tailscale up` on each one, log in
through your identity provider, and they're peers. Underneath
it's WireGuard (kernel module on Linux, in-process userspace on
macOS / Windows / iOS / Android), which gives you AEAD-encrypted,
single-RTT, public-key-authenticated tunnels with no shared
secret to rotate. The control plane (open-source as
[`headscale`](https://github.com/juanfont/headscale) if you want
to self-host) only passes node public keys + ACL — it never sees
your traffic. The killer features layered on top are MagicDNS
(`ssh laptop` instead of `ssh 100.74.12.5`), Tailscale SSH
(SSH auth handled by the tailnet, no `~/.ssh/authorized_keys` to
maintain), Subnet routing (one node advertises `10.0.0.0/24` and
the whole tailnet can reach the home LAN behind it), and Exit
nodes (route every byte of traffic through one tailnet member —
poor man's VPN with hosts you actually own).

## Install

```bash
# macOS
brew install --cask tailscale

# Linux (one-shot installer, configures the package repo)
curl -fsSL https://tailscale.com/install.sh | sh

# verify + log in
tailscale version           # 1.96.4
sudo tailscale up           # opens a browser to authenticate
tailscale ip -4             # your tailnet IPv4
tailscale status            # all peers + their IPs / online state
```

The free "Personal" plan covers up to 100 devices and 3 users —
plenty for a one-person dev fleet. Self-host the control plane
with `headscale` if you want zero dependence on a managed service.

## One Concrete Example

```bash
# 1. install + log in on three boxes (laptop, cloud VM, home NAS)
sudo tailscale up
# (open browser, auth, name device)

# 2. addressable by name from anywhere
ssh laptop                              # MagicDNS resolves to 100.x.y.z
curl http://nas:8080/health             # private HTTP across the internet
rsync -av photos/ laptop:~/photos/      # works from any tailnet member

# 3. expose home LAN to the tailnet (NAS routes 192.168.1.0/24)
sudo tailscale up --advertise-routes=192.168.1.0/24
# in admin console: approve the advertised route once
ping 192.168.1.1                        # router reachable from cloud VM

# 4. route all internet traffic through home (exit node)
# on home NAS:
sudo tailscale up --advertise-exit-node
# on laptop, when on hostile wifi:
sudo tailscale up --exit-node=nas --exit-node-allow-lan-access

# 5. SSH without ~/.ssh/authorized_keys
sudo tailscale up --ssh                 # on the destination
ssh laptop                              # auth handled by tailnet identity

# 6. one-shot file send between devices
tailscale file cp ./report.pdf laptop:
tailscale file get .                    # on the receiving box
```

## Niche It Fills

**A flat private network across machines you own, without the
overhead of running a real VPN concentrator.** The classic
WireGuard story is "edit `/etc/wireguard/wg0.conf` on every node,
hand-allocate IPs, distribute public keys, hope the NAT works."
tailscale automates every step of that — identity, key rotation,
NAT traversal, DNS, ACLs — and treats the result as one logical
network. For an LLM-CLI / agent operator who wants their `home`
machine reachable from a laptop on a coffee-shop wifi without
opening a single inbound port on the home router, this is the
"just works" answer.

## Vs Already Cataloged

- **Vs [`mosh`](../mosh/):** mosh is a *resilient SSH transport*
  over UDP (survives roaming + sleep). tailscale is the *network
  layer underneath*. They compose — `mosh laptop` over a
  tailnet works fine and gets you both benefits.
- **Vs [`sshuttle`](../sshuttle/):** sshuttle is "VPN-over-SSH"
  for one bastion host (route subnets through an SSH session,
  no admin on the remote needed). tailscale needs to be installed
  on every endpoint but gives you a *real* mesh — not a hub-and-
  spoke through one bastion. Use sshuttle for ad-hoc /
  one-bastion situations; tailscale for a fleet you control.
- **Vs [`bore`](../bore/) / [`ngrok`](https://ngrok.com)-style
  tunnels:** those punch *one inbound TCP port* through to a
  local service for *public* access. tailscale gives a *whole
  private network* between authenticated peers. tailscale's
  Funnel feature does the public-tunnel job too if you need
  both shapes from one tool.
- **Vs raw WireGuard (`wg-quick`):** tailscale *is* WireGuard at
  the data plane; it adds the control plane (identity, NAT
  traversal, key rotation, ACLs, DNS) that vanilla WireGuard
  expects you to operate yourself. Pick raw WireGuard for
  fully-on-prem zero-trust to a vendor; tailscale otherwise.

## Caveats

- The default control plane is the **managed
  `controlplane.tailscale.com`** — it never sees your packet
  payloads (those go peer-to-peer over WireGuard) but it does
  see node metadata, ACL evaluation, and DERP-relayed bytes
  *encrypted-but-routed-through-them* when direct NAT traversal
  fails. Self-host with [`headscale`](https://github.com/juanfont/headscale)
  if even that metadata is a concern.
- License is **BSD-3-Clause for the open-source `tailscale` and
  `tailscaled` binaries in this repo**; the macOS / iOS / Windows
  GUI clients and the hosted control-plane service are separate
  proprietary products. The CLI / daemon you install via
  `brew install tailscale` (Homebrew formula, not the cask) or
  the Linux package repo is the BSD-licensed build.
- Exit-node mode disables the OS's default route while active —
  if `tailscaled` crashes mid-flight you can lose connectivity
  until the route is restored (`sudo tailscale down` /
  `sudo tailscale up --exit-node=`). Test on a host you can
  recover before relying on it.
- Tailscale SSH replaces public-key auth with tailnet identity —
  great for ergonomics, but it means losing access to your
  identity provider (locked-out Google / GitHub account) means
  losing SSH access to every node where Tailscale SSH is the
  only path. Keep a break-glass `~/.ssh/authorized_keys` entry
  on at least one node.
- ACL changes are **HuJSON in the admin console** (or via the
  API) and apply tailnet-wide on save — a typo in `"src"` /
  `"dst"` can lock peers out of each other. Tailscale offers a
  preview / dry-run; use it.
