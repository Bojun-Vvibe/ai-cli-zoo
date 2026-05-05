# nexttrace

> **Modern open-source `traceroute` / `mtr` replacement
> with built-in IP geolocation, ASN annotation, and
> map output** — every hop is enriched with
> city / country / ASN / ISP / latency in one pass, no
> separate `whois` round-trips. Pinned to **v1.6.4**
> ([LICENSE](https://github.com/nxtrace/NTrace-core/blob/main/LICENSE),
> GPL-3.0).

Source: <https://github.com/nxtrace/NTrace-core>

## TL;DR

`nexttrace` (binary: `nexttrace`, sometimes packaged as
`nxtrace`) is a single Go binary that probes the path
to a destination using ICMP / TCP / UDP (kernel raw
sockets, no `libpcap` dependency) and, for every hop,
calls a free geolocation API to annotate it with
`AS<n>` + ISP name + city + country + lat/lon — so the
output of `nexttrace one.one.one.one` is not just a
column of IPs and RTTs but a routed-path narrative
("hop 4: AS9808 China Mobile, Beijing → hop 7: AS3257
GTT, Frankfurt → hop 9: AS13335 Cloudflare,
Amsterdam"). Subcommands cover the usual investigative
shapes: `nexttrace -T 443 host` for TCP traceroute
through port-blocking middleboxes, `nexttrace -M host`
for `mtr`-style continuous probing with rolling stats,
`nexttrace --map host` opens a browser tab plotting the
hops on a world map, `nexttrace --report host` produces
a one-shot Markdown report suitable for an outage
ticket, and `--data-provider` swaps among IPInfo /
LeoMoeAPI / IP.SB / IPAPI / disable-geolocation for the
"my probe is rate-limited" or "I cannot leak this IP
to a third-party API" case. Pure user-space probes
(setcap cap_net_raw on Linux, no kernel module), works
behind NAT, and IPv6 is first class.

## When to use

- You debug **cross-region / cross-AS network paths**
  and the question is *who owns this hop* — bare
  `traceroute` gives you an IP; nexttrace gives you
  ASN + ISP + geography in the same line.
- You need **TCP traceroute** to a specific port
  because the path drops ICMP — `nexttrace -T 443` is
  the one-flag answer.
- You want a **shareable artifact** of a network
  investigation — `--report` prints clean Markdown
  and `--map` produces a browser-renderable map.

## When NOT to use

- You need a **continuously running long-term path
  monitor with alerting** — that is the niche of
  [`trippy`](../trippy/) (TUI) or `smokeping` /
  Prometheus blackbox-exporter (server-side); nexttrace
  is investigative, not a monitoring daemon.
- You operate in a **fully air-gapped network** and
  cannot make any external HTTP call — disable the
  geolocation lookups (`--data-provider disable-geoip`)
  or fall back to vanilla `traceroute` / `mtr`.
- You only ever care about the **per-hop loss + RTT
  histogram** — [`mtr`](../mtr/) is smaller, ubiquitous,
  and packaged everywhere.

## Install

```bash
# Homebrew
brew install nexttrace

# upstream install script
curl -fsSL nxtrace.org/nt | bash

# Go install
go install github.com/nxtrace/NTrace-core@latest

# prebuilt binaries
# https://github.com/nxtrace/NTrace-core/releases

# Linux: grant raw-socket capability so it runs unprivileged
sudo setcap cap_net_raw+ep "$(command -v nexttrace)"

# verify
nexttrace --version    # NextTrace v1.6.4
```

## Basic usage

```bash
# default: ICMP traceroute with geo + ASN annotation
nexttrace one.one.one.one

# TCP traceroute to a specific port (path drops ICMP)
nexttrace -T -p 443 example.com

# mtr-style continuous probing with rolling stats
nexttrace -M example.com

# render the path on a world map in your browser
nexttrace --map example.com

# Markdown report for an outage ticket
nexttrace --report example.com > path.md

# pick a different geo provider (or disable entirely)
nexttrace --data-provider IPInfo example.com
nexttrace --data-provider disable-geoip example.com

# IPv6
nexttrace -6 google.com
```

## Pairs with

- [`mtr`](../mtr/) / [`trippy`](../trippy/) — the TUI
  / continuous-monitoring peers; nexttrace is the
  one-shot enriched-output peer.
- [`dog`](../dog/) / [`doggo`](../doggo/) — DNS
  lookups for the targets you are about to trace.
- [`bandwhich`](../bandwhich/) — local per-process
  bandwidth view; nexttrace is the path view.
- [`gping`](../gping/) — graphical ping for steady-
  state latency to the hops nexttrace identifies.
