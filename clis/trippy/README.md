# trippy

- **Repo:** https://github.com/fujiapple852/trippy
- **Version:** v0.13.0 (latest stable, 2025)
- **License:** Apache-2.0 ([LICENSE](https://github.com/fujiapple852/trippy/blob/master/LICENSE))
- **Language:** Rust
- **Install:** `brew install trippy` · `cargo install --locked trippy` · `pacman -S trippy` · static binaries on the GitHub release page · binary name is `trip`

## What it does

`trippy` (binary: `trip`) is a network diagnostics TUI that fuses
`traceroute` and `mtr` into a single tool, then adds the things both have
been missing for twenty years. Like `mtr` it sends repeated probes along
the path to a target and shows live per-hop loss / latency stats; unlike
`mtr` it speaks **ICMP, UDP, and TCP** probes (so you can trace past
firewalls that drop ICMP by sending TCP SYNs to port 443, which is what
real traffic looks like), supports **IPv4 and IPv6** in a single binary,
runs **multi-target traces in parallel** in tabbed panes, and can
optionally use the **Paris/Dublin** flow-identifier trick to keep all
probes on the same ECMP path so you get a stable picture instead of a
phantom-route mess. The TUI shows the classic per-hop table (hop number,
IP, hostname, AS number via Team Cymru DNS, loss%, last/avg/best/worst/std
RTT, and a small in-line latency sparkline) plus an optional **flow chart
view** that visualizes ECMP path divergence, a **map view** when you load
a GeoIP MMDB, and a **DNS lookup column** that resolves PTRs in the
background without blocking probes. Output can be exported to JSON, CSV,
or a pretty Markdown table for pasting into a ticket. Configuration lives
in `$XDG_CONFIG_HOME/trippy/trippy.toml` and lets you pin probe protocol,
target port, max hops, refresh rate, and theme.

## When to pick it / when not to

Pick `trippy` whenever the question is "where on the path is this
connection breaking" and `ping` is not enough. It is the right tool for
debugging high-latency or lossy connections to a specific host, for
proving "the issue is at hop 7 in $TRANSIT_PROVIDER, not on our
edge", for tracing past corporate firewalls that black-hole ICMP (use
`-p tcp -P 443`), and for diagnosing ECMP-related "sometimes fast,
sometimes slow" weirdness with the Paris probe mode. The AS-number
column is genuinely useful — when you can see "loss starts at the hop
where we leave AS13335 and enter AS174", the conversation with the
upstream NOC gets a lot shorter. Pair it with `dig` / `kdig` for the
DNS half of the puzzle, with [`bandwhich`](../bandwhich/) when the
question shifts from "where is the path broken" to "what is using my
bandwidth", with [`dog`](../dog/) or [`doggo`](../doggo/) for modern
DNS lookups, and with [`gping`](../gping/) when you only need a graphed
ping to one host without the full path breakdown.

Skip it for one-shot "is this host up" checks — `ping` is faster to type
and reach. Skip it inside container networks where the ICMP / raw-socket
capabilities are dropped — without `CAP_NET_RAW` (or running as root, or
the macOS `setuid` install that Homebrew sets up) probes will fail to
send. Skip it for application-layer debugging — once you have confirmed
the path is clean, the bug is above L4 and you want `curl -v`, `httpie`,
[`xh`](../xh/), or a real APM trace, not another packet probe. Skip it
when you need long-running historical metrics — trippy is a live
diagnostic, not a time-series collector; for 24/7 path monitoring use
SmokePing, Prometheus blackbox-exporter, or a hosted equivalent. Be
aware on macOS that the Homebrew formula installs the binary setuid-root
to grant raw-socket access; that is the standard `mtr` / `traceroute`
posture but worth noting in security-conscious environments.

## Example invocations

```bash
# Default: ICMP trace to a host, live TUI
trip example.com

# TCP probes to port 443 — works through firewalls that drop ICMP
trip -p tcp -P 443 example.com

# UDP probes (classic traceroute behaviour) on a custom port
trip -p udp -P 33434 example.com

# Trace multiple targets in parallel, each in its own tab
trip example.com cloudflare.com 1.1.1.1

# Force IPv6
trip -6 example.com

# Paris-traceroute mode: keep all probes on the same ECMP flow
trip --multipath-strategy paris example.com

# Increase probe rate and shorten max hops for a fast LAN trace
trip -i 50ms -m 12 192.168.1.1

# Export a JSON report after N rounds (useful for tickets / CI checks)
trip --mode silent -c 50 --report-format json example.com > trace.json

# Pretty Markdown report for pasting into an incident doc
trip --mode silent -c 30 --report-format markdown example.com
```
