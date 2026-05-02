# iperf3

> **The canonical client/server tool for measuring achievable
> network bandwidth between two hosts** — start `iperf3 -s` on
> one machine, run `iperf3 -c <host>` on the other, and get a
> precise per-second throughput report (TCP or UDP, IPv4 or
> IPv6, single or parallel streams) along with retransmits, RTT,
> congestion-window snapshots, and JSON output for scripting.
> Pinned to **v3.21**
> ([LICENSE](https://github.com/esnet/iperf/blob/master/LICENSE),
> BSD-3-Clause-LBNL).

Source: <https://github.com/esnet/iperf>

## TL;DR

`iperf3` is what you reach for when you need to know "how fast
can these two hosts actually move bytes between each other,
right now?" — independent of the application protocol, the
disk, the database, or the CDN. It is a complete rewrite of the
older `iperf2` (also still maintained, separately) by ESnet at
LBNL, with a cleaner code base, JSON output, and modern TCP
diagnostics. It saturates a link with synthetic traffic and
reports goodput, jitter, packet loss (UDP), retransmits and
congestion-window evolution (TCP/Linux), all per-interval and
totals. It is the standard answer to "is my 10 GbE actually
10 GbE", "why is my VPN slow", "did this kernel upgrade hurt
network throughput", and "what is the real bandwidth between
these two cloud regions".

## Install

```bash
# Homebrew (macOS / Linux)
brew install iperf3

# Linux package managers
# Debian/Ubuntu: apt install iperf3
# Fedora/RHEL:   dnf install iperf3
# Arch:          pacman -S iperf3
# Alpine:        apk add iperf3
# openSUSE:      zypper install iperf3

# FreeBSD
pkg install iperf3

# Build from source (any POSIX with a C compiler)
curl -LO "https://github.com/esnet/iperf/releases/download/3.21/iperf-3.21.tar.gz"
tar -xzf iperf-3.21.tar.gz
cd iperf-3.21
./configure && make && sudo make install

# Windows: use the maintained binaries from iperf.fr or run via
# WSL2; upstream esnet/iperf does not ship Windows builds.

# verify
iperf3 --version    # iperf 3.21 (cJSON 1.7.x) ...
```

A single binary, no daemon required (the server is just
`iperf3 -s` running in the foreground or as a one-shot).

## License

BSD-3-Clause-LBNL — see
[LICENSE](https://github.com/esnet/iperf/blob/master/LICENSE).
A BSD variant with a Lawrence Berkeley National Laboratory
notice clause; permissive, fine for embedding in commercial
products and shipping inside Docker images.

## One Concrete Example

```bash
# 1. on host A (the server) — listens on TCP/5201 by default
iperf3 -s

# 2. on host B (the client) — TCP, 10 seconds, 1 stream
iperf3 -c hostA
# [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
# [  5]   0.00-1.00   sec   112 MBytes   941 Mbits/sec    0    1.43 MBytes
# ...
# [SUM]  0.00-10.00  sec  1.10 GBytes   941 Mbits/sec    0   sender

# 3. parallel streams to saturate a link with a single flow that
#    can't fill the bandwidth-delay product
iperf3 -c hostA -P 8 -t 30

# 4. UDP test with target rate 500 Mbit/s, measure jitter + loss
iperf3 -c hostA -u -b 500M -t 30
# Server output includes: Jitter, Lost/Total Datagrams, %loss

# 5. reverse mode: server sends, client receives
#    (useful when the client is behind NAT and you want
#    download-direction throughput from the public host)
iperf3 -c hostA -R -t 30

# 6. JSON output for scripting / dashboards
iperf3 -c hostA -t 10 --json | jq '.end.sum_received.bits_per_second'
# 9.412345e+08

# 7. bind to specific source interface (multi-NIC host)
iperf3 -c hostA -B 10.0.1.5 -t 10

# 8. TCP MSS / window tuning for long-fat-network measurement
iperf3 -c hostA -w 4M -t 60

# 9. authenticated server (3.x feature) — RSA pubkey auth so a
#    public iperf3 server can rate-limit by user
iperf3 -s --rsa-private-key-path server.key --authorized-users-path users.csv
iperf3 -c hostA --rsa-public-key-path server.pub --username alice
```

## Niche It Fills

**Active, synthetic-traffic bandwidth measurement between two
cooperating endpoints.** This is a different question from
"what is my web app's response time" (use `curl --write-out` /
`oha` / `vegeta`) or "what does my network look like
end-to-end" (use `mtr` / `traceroute`) — `iperf3` answers
specifically: with both ends under your control, how many bits
per second can the path actually carry? It's the layer-4
equivalent of `dd if=/dev/zero of=/dev/null` for disks: a pure
throughput probe. Cloud providers, ISPs, datacenter operators,
and home-lab tinkerers all reach for it for the same reason.

## Why use it

Three things `iperf3` does that explain why it remains the
default after 10+ years and why simpler "speedtest" tools don't
displace it for engineering work:

1. **Two-sided, independent of the public internet.** Public
   speedtests measure your path to *their* server, which is
   often the bottleneck and usually rate-limited. With
   `iperf3 -s` on one of your hosts, you measure the actual
   path between your hosts — across a VPN tunnel, between two
   AZs, across a leased line, between a NAS and a workstation
   — with no third-party variability.
2. **Rich, structured per-second telemetry.** Each interval
   reports throughput, retransmits, and (on Linux/macOS as of
   3.21) the TCP congestion window. The `--json` flag emits the
   whole report as machine-readable JSON, so you can pipe into
   Prometheus textfile collectors, Grafana, or your own
   dashboards. UDP mode adds jitter and per-datagram loss
   counters — essential for VoIP / video / game-server pathing.
3. **Knobs that match how networks actually behave.** Parallel
   streams (`-P`), reverse direction (`-R`), bidirectional
   (`--bidir`), socket buffer size (`-w`), MSS (`-M`),
   congestion control selection (`-C cubic|bbr|reno`), DSCP
   marking (`-S`), and zero-copy (`-Z`) are all single flags.
   You don't need a custom traffic generator to model a
   bandwidth-delay product or to test a specific QoS class.

For an LLM-CLI workflow that diagnoses network-bound steps —
"my data sync is slow", "the model upload is taking forever" —
`iperf3 -c <peer> -t 10 --json` between the two endpoints
gives the agent a hard ground-truth number for available
bandwidth, before it spends time guessing at application-layer
explanations.

## Vs Already Cataloged

- **Vs `bombardier` / `vegeta` / `oha` / `k6` (HTTP load
  generators):** Those tools answer "how does my HTTP service
  perform under load" — they exercise app-layer code paths
  (TLS, parsing, handlers, downstream calls). `iperf3` answers
  "what is the underlying network capable of" — pure
  TCP/UDP throughput, no application stack involved. Use HTTP
  load tools to find slow handlers; use `iperf3` to confirm
  the network isn't the bottleneck before you start blaming
  the handlers.
- **Vs `mtr` / `traceroute` (path analysis):** Those tools map
  the path and per-hop latency / loss; they do not measure
  achievable throughput. `iperf3` doesn't show you the path,
  but tells you what bandwidth you can actually push end-to-
  end. They are complements: `mtr` to find which hop is lossy,
  `iperf3` to quantify what that does to your throughput
  ceiling.
- **Vs `netcat` + `pv` for ad-hoc throughput tests:**
  `nc -l 9999 > /dev/null` on one side, `pv < bigfile | nc
  host 9999` on the other works in a pinch and needs no extra
  install. But it gives you one aggregate number, no per-
  interval breakdown, no UDP jitter, no parallel-stream
  mode, no JSON, no congestion-window data, and the disk read
  on the sender often becomes the bottleneck. `iperf3`
  generates traffic in memory and is purpose-built for the
  measurement.

## Caveats

- **`iperf3` is single-threaded per test session.** A single
  TCP stream on a >10 Gbps link will often be CPU-bound on one
  core long before it saturates the NIC. Use `-P 4` or `-P 8`
  to spread across cores when measuring high-bandwidth paths,
  or you'll under-report the link capacity.
- **`iperf3` is not protocol-compatible with `iperf2`.** They
  are separate forks with separate code bases and separate
  features (iperf2 has multicast, multi-client server mode,
  `--isochronous`; iperf3 has JSON output, authentication,
  cleaner code). Run the same major version on both ends.
- **Default port 5201 must be reachable.** Behind NAT or
  restrictive firewalls, the server-binding side needs the port
  forwarded. `--port`/`-p` lets you change it; `-R` lets the
  *server* push so the client only needs an outbound
  connection.
- **UDP mode does not auto-discover path MTU.** Default UDP
  packet size is 1460 bytes; on links with smaller MTU
  (tunnels, some VPNs) you must set `-l` explicitly to avoid
  fragmentation skewing the loss numbers.
- **Old `iperf3` versions had a known per-test memory leak in
  the daemon-style server (long-running `-s` accumulating
  sessions over weeks). Fixed in modern releases (3.18+); if
  you run a public test server, restart `iperf3 -s` between
  sessions or use `-1` (one-off) mode under a process
  supervisor.
- **Don't run a public iperf3 server without auth.** A bare
  `iperf3 -s` on the public internet is an open bandwidth-
  burning amplifier. Use `--rsa-private-key-path` +
  `--authorized-users-path` (3.x) and rate-limit at the
  firewall.
