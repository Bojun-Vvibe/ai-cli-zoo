# sshuttle

## What it does
A **transparent IP-over-SSH tunnel** ("poor person's VPN") written in
Python: launches with `sshuttle -r user@bastion 10.0.0.0/8`, opens a
single SSH connection to a remote host that has a Python interpreter
(no root, no special server software, no daemon to install), and
transparently forwards every TCP connection from the local machine
whose destination matches one of the supplied subnets through the
remote side — the remote acts as an exit node and resolves the target
IPs from its own network position. Crucially, sshuttle does **not**
encapsulate UDP-in-TCP at the IP layer (no TCP-meltdown problem like
real-VPN-over-TCP fallbacks); it uses `iptables` / `nftables` /
`pf` (macOS) NAT redirection or a TPROXY rule to grab matched flows
and re-emits them as fresh TCP streams on the remote, multiplexed
inside the SSH channel. DNS can be optionally tunneled via `--dns`
(remote does the resolution — useful for hitting private Route53 /
internal CoreDNS zones from a laptop), individual subnets can be
excluded with `-x`, and `--auto-hosts` populates `/etc/hosts` with
remote hostnames discovered via reverse DNS so corp short-names
resolve locally. Daemon mode (`-D` + PID file), sudoers presets
shipped in `/usr/share/sshuttle/sudoers/`, and a recent rewrite of
broken-pipe / EPIPE handling (v1.3.2, 2025-08) make it stable for
multi-day always-on sessions.

## Why it's interesting
Different shape from a real VPN — WireGuard, OpenVPN, Tailscale,
Nebula, Zerotier — which require server-side software, kernel
modules or root-installed userspace tunnels, account provisioning,
key rotation, and (for corporate VPNs) IT-department buy-in. Different
shape from `ssh -D <port>` SOCKS5 (per-application proxy config
required, breaks anything that doesn't speak SOCKS — hello, native
gRPC and most database drivers; doesn't handle DNS), from `ssh -L`
port-forwards (one-port-at-a-time, doesn't scale to "everything in
this VPC"), from `chisel` / `frp` (excellent reverse-tunnel
servers, but you have to run the server side), and from `cloudflared
access tcp` (great when you've already bought into Cloudflare Zero
Trust, but vendor lock-in). sshuttle is the *zero-server-install,
just-needs-Python-on-the-far-end, transparent-by-subnet, your-corp-
SSH-bastion-already-supports-it* shape: pick it specifically for
"my laptop is on a coffee-shop wifi and I need to `psql` into a
private RDS instance behind a bastion right now", for routing only
internal subnets through the bastion while keeping public traffic on
the local link (split tunneling done right), and for environments
where installing or operating a VPN appliance is politically or
practically impossible. Do **not** pick it as a general-purpose
privacy VPN (it doesn't tunnel arbitrary UDP — Wireguard / OpenVPN /
Tailscale are the right answer), as a replacement for service-mesh /
zero-trust networking at scale, or in any setting where your AGENTS
rules forbid offensive-security tooling — sshuttle itself is a
legitimate sysadmin tool used for exactly this access pattern, but
do not pair it with any of the banned categories.

## Niche category
Transparent SSH-tunneled VPN — Python, kernel-NAT-based "poor
person's VPN" that piggybacks on an existing SSH bastion with no
server-side install, ideal for split-tunneling private subnets to
a developer laptop.

## Repo
https://github.com/sshuttle/sshuttle

## Version pinned
`v1.3.2` (latest tagged release as of 2026-05-02, published
2025-08-10)

## License
- SPDX: `LGPL-2.1-or-later`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install sshuttle

# pip (any platform with Python 3.9+)
pip install --user sshuttle==1.3.2

# Debian / Ubuntu
apt-get install sshuttle

# From source
git clone --branch v1.3.2 https://github.com/sshuttle/sshuttle.git
cd sshuttle && pip install .
```

## Usage examples
```sh
# Route the entire 10.0.0.0/8 corp range through bastion.example.com
sudo sshuttle -r user@bastion.example.com 10.0.0.0/8 172.16.0.0/12

# Tunnel everything except your local LAN and the SSH session itself
sudo sshuttle -r user@bastion 0/0 -x 192.168.0.0/16 -x bastion.example.com

# Also tunnel DNS so private Route53 / internal zones resolve correctly
sudo sshuttle --dns -r user@bastion 10.0.0.0/8

# Daemon mode with PID file (multi-day always-on session)
sudo sshuttle -D --pidfile /var/run/sshuttle.pid \
  -r user@bastion 10.0.0.0/8

# Auto-populate /etc/hosts with discoverable remote hostnames
sudo sshuttle --auto-hosts -r user@bastion 10.0.0.0/8
```

## Date added
2026-05-02
