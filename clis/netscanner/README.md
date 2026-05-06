# netscanner

> **netscanner** — Chleba/netscanner, a TUI network diagnostic
> dashboard that combines local-interface inventory, LAN host
> discovery (ARP scan), Wi-Fi scanning, ping sweeps, traceroute, and
> per-port reachability into one keyboard-driven Ratatui interface.
> Pinned to **v0.6.41**, MIT — license file:
> [LICENSE](https://github.com/Chleba/netscanner/blob/main/LICENSE).

Source: <https://github.com/Chleba/netscanner>

## TL;DR

```bash
sudo netscanner            # most features need raw socket / ARP
```

The TUI opens with a tab bar across the top — `Discovery`,
`Packetdump`, `Ports`, `Traffic`, `Wifi` — each backed by a
real-time pane. Discovery does an ARP sweep over your local /24,
shows live IP / MAC / vendor (via the `mac-oui-lookup` table) and
hostname (via reverse-DNS) for each responder, and updates as new
hosts join the LAN. Ports lets you tab-drill into one host and
probe a configurable port list with reachability latency. Traffic
plots in/out bytes per interface as a sparkline. Wifi enumerates
nearby APs with SSID / BSSID / signal / channel / encryption.
Packetdump is a tcpdump-shaped frame log filterable by interface.

Everything is keyboard-driven (`Tab` switches panes, `j/k` moves
selection, `Enter` drills in, `q` exits) and the rendering is a
true Ratatui dashboard with proportional column widths, no flicker
on updates, and Unix-pipe-friendly snapshots.

## What it actually solves

The "I'm on a new network and I want to see what's on it" workflow
that traditionally requires three or four tools:

- `ip a` / `ifconfig` for local interface state
- `arp -a` or `arp-scan` for LAN inventory
- `iwlist scan` / `airport -s` / `nmcli dev wifi` for Wi-Fi enumeration
- `nc -zv` or `nmap -F` for port reachability on one host
- `iftop` / `nethogs` for live throughput

netscanner unifies them into one TUI so the LAN sweep, the Wi-Fi
list, the port probe on the interesting host, and the throughput
plot all live in panes you can flip between with `Tab`. It's the
"diagnostic dashboard" shape, deliberately defensive — discovery
and reachability for *your own* networks, not exploit primitives
or vulnerability detection.

## Install

```bash
# Cargo (most platforms)
cargo install netscanner

# macOS / Linux pre-built binaries
# Download from https://github.com/Chleba/netscanner/releases

# Arch Linux
yay -S netscanner

# Run with sudo for raw-socket / ARP / packet-capture features
sudo netscanner
```

Note that ARP discovery, packet dump, and Wi-Fi scan all need
elevated privileges (raw sockets / `CAP_NET_RAW` / equivalent
macOS entitlement). Without sudo you still get interface inventory
and TCP-connect port checks but the LAN sweep and packet pane go
empty.

## Why orthogonal to the existing zoo

The zoo has many network-utility CLIs but the niche is precise:

- [`bandwhich`](../bandwhich/) is the closest neighbor on the
  *throughput* axis — it shows current per-process / per-connection
  bandwidth for the **local** host. **bandwhich wins on per-process
  attribution** (which app is eating the uplink). **netscanner wins
  on LAN inventory** (who else is on this Wi-Fi). They coexist:
  bandwhich for "what on my laptop is using the network",
  netscanner for "what other devices are on this network".
- [`sniffnet`](../sniffnet/) is a packet-statistics dashboard with a
  slightly different lens — it focuses on traffic *classification*
  (per-protocol / per-host volume from the local NIC) and exports.
  netscanner adds discovery (ARP sweep) and Wi-Fi enumeration on
  top; sniffnet wins on the classification depth.
- [`bandwidth`](../bandwhich/) family covers throughput; [`iftop`](../iftop/),
  [`nethogs`](../nethogs/), [`vnstat`](../vnstat/) are the classic
  per-interface / per-process monitors. Same throughput axis.
- [`mtr`](../mtr/), [`trippy`](../trippy/), [`nexttrace`](../nexttrace/)
  are *path* tools — traceroute / pathping shape, latency per hop
  to one destination. Different verb than netscanner's LAN sweep.
- [`dog`](../dog/), [`doggo`](../doggo/), [`dnslookup`](../dnslookup/)
  are DNS-resolution tools, not network discovery.
- [`gping`](../gping/) plots ping latency per target — narrower than
  netscanner's full dashboard.
- [`termshark`](../termshark/) is the full Wireshark-in-a-TUI for
  deep packet inspection. **termshark wins on capture analysis**;
  netscanner wins on *live multi-pane LAN/Wi-Fi/port discovery* with
  none of the capture-file workflow overhead.
- [`impala`](../impala/) is a Wi-Fi-specific TUI manager (connect to
  APs, manage profiles). **impala wins as a Wi-Fi connection manager**;
  netscanner is read-only enumeration in service of the broader
  diagnostic dashboard.

In practice: netscanner is the "open one TUI, see the whole LAN
posture" tool for the on-call / new-network / home-lab setup
moment. Use bandwhich for per-process throughput, termshark for
deep packet inspection, impala when you actually need to manage
Wi-Fi connection state.

## Pairs with

- [`bandwhich`](../bandwhich/) — per-process throughput on the local
  host
- [`sniffnet`](../sniffnet/) — protocol-classification dashboard
- [`trippy`](../trippy/) / [`mtr`](../mtr/) — path latency to a
  specific destination once netscanner identifies the interesting
  remote
- [`dog`](../dog/) / [`doggo`](../doggo/) — DNS resolution against
  the discovered hosts
- [`termshark`](../termshark/) — drop into deep packet inspection on
  one interface when netscanner's packetdump pane isn't enough
- [`impala`](../impala/) — flip from netscanner's Wi-Fi enumeration
  to actually connecting to the chosen AP

## Caveats

- Most useful panes need root / sudo (raw sockets, ARP, Wi-Fi scan,
  packet dump). Without it, only interface inventory and TCP-connect
  port probes work.
- The LAN sweep is ARP-based, so it's bounded to your local L2
  segment — won't see hosts behind another router.
- Wi-Fi enumeration uses platform-specific backends (Linux
  `nl80211` / macOS CoreWLAN / Windows native wifi) — feature parity
  varies; check the README for current per-platform support matrix.
- This is a **read-only diagnostic** tool, not a vulnerability
  scanner / exploitation framework. It doesn't fingerprint OS,
  doesn't probe for CVEs, doesn't send anything beyond ARP /
  ICMP / TCP-connect. Use only on networks you administer or have
  written permission to scan.
- Project is single-maintainer with active cadence — pin the version
  in any automation that relies on the JSON-export shape (still
  evolving).
- Older Linux kernels may need `CAP_NET_RAW` capability set
  explicitly via `setcap cap_net_raw,cap_net_admin=eip
  $(which netscanner)` instead of running as root.
