# iftop

> **Real-time `top` for network bandwidth, per host pair** — a
> ~150 KB C utility that opens a libpcap capture on a chosen
> interface and renders a continuously-refreshing ncurses table
> of the active TCP/UDP/ICMP flows ranked by current bandwidth,
> with three rolling averages (last 2 s / 10 s / 40 s) per
> direction and a footer showing send / receive / total rates
> for the box as a whole. Pinned to **v1.0pre4**
> ([LICENSE](https://code.blinkace.com/pdw/iftop/src/branch/master/COPYING),
> GPL-2.0-or-later).

Source: <https://code.blinkace.com/pdw/iftop> (canonical upstream)
and <https://www.ex-parrot.com/pdw/iftop/> (long-standing
homepage with release tarballs).

## TL;DR

`iftop` is the network-observability primitive that fits in your
muscle memory next to `top` and `iotop`: one command, one
interface argument, and an immediate ranked view of *who is
talking to whom on this box right now and at what rate*.
`sudo iftop -i en0` opens a libpcap capture, builds a
continuously-updated table keyed by `(local-host, remote-host)`,
and renders it with arrow-bars showing send vs receive direction
and three columns of moving averages so a flow that just spiked
is visually distinct from a flow that has been steady for
minutes. Interactive single-key commands toggle DNS resolution
(`n`), port resolution (`p`), per-source (`s`) / per-destination
(`d`) port columns, sort key (`T` / `2` / `3` / `<` / `>`), and
filter by BPF expression (`f`) — the same expression syntax
`tcpdump` uses, so `f` then `port 443 and not host 10.0.0.5`
narrows the table without restarting. The whole tool is one C
binary plus libpcap + ncurses; no daemon, no config file
required, no kernel module.

## Install

```bash
# Homebrew (macOS — needs sudo to actually capture)
brew install iftop

# Debian / Ubuntu
sudo apt install iftop

# Fedora / RHEL
sudo dnf install iftop

# Arch
sudo pacman -S iftop

# Alpine
sudo apk add iftop

# from source (any Unix with libpcap-dev + libncurses-dev)
git clone https://code.blinkace.com/pdw/iftop.git
cd iftop && ./configure && make && sudo make install

# verify
iftop -h | head -1     # iftop: display bandwidth usage on an interface
```

Capturing packets requires `CAP_NET_RAW` on Linux or root on
macOS — practically every invocation is `sudo iftop …`. The
`setcap cap_net_raw,cap_net_admin=eip $(which iftop)` trick on
Linux lets a non-root user run it without `sudo` if you prefer.

## License

GPL-2.0-or-later — see
[COPYING](https://code.blinkace.com/pdw/iftop/src/branch/master/COPYING).
Binary redistribution is fine; downstream forks owe source under
the same licence.

## One Concrete Example

```bash
# 1. simplest run — pick an interface, watch the table fill
sudo iftop -i en0

# 2. machine-readable text mode — print 5 samples then exit
#    (great for cron-driven "what was the network doing at 03:14")
sudo iftop -i en0 -t -s 5 -L 20 > /tmp/net-$(date +%s).txt

# 3. show port columns, no DNS lookups (fast in DNS-blackholed envs)
sudo iftop -i en0 -nNP

# 4. filter to one service, sort by 10 s average descending
sudo iftop -i en0 -f "port 443" -o 10s

# 5. CIDR-scoped view — "how much am I shovelling to S3 right now"
sudo iftop -i en0 -F 52.216.0.0/15 -nN

# 6. find the noisy pod on a k8s node — capture on the bridge,
#    skip name resolution, show source ports
sudo iftop -i cni0 -nNP -B          # -B reports bytes/sec, not bits/sec

# 7. tcpdump-style BPF for "anything outbound that isn't DNS or NTP"
sudo iftop -i en0 -f "outbound and not (port 53 or port 123)"

# 8. inside the TUI:
#    n  toggle DNS              p  toggle port column
#    s  toggle source-port      d  toggle dest-port
#    t  cycle line layout       T  show cumulative totals
#    1/2/3  sort by 2s/10s/40s avg
#    <  sort by source addr     >  sort by dest addr
#    j/k  scroll the flow list  P  pause display
#    f  edit BPF filter         l  edit screen filter
#    q  quit
```

## Niche It Fills

**The `top` for bandwidth, sized for one box.** When a server is
suddenly slow and the question is "is the network the
bottleneck, and if so, *who* is the network being used by", the
useful answer is a live ranked table of flows by current rate —
not a five-minute Prometheus rollup, not a packet capture you
have to load into Wireshark, not an SNMP poll on the switch.
`iftop` is the lowest-friction tool that produces exactly that
view: a single binary you already have in your distro's repos, a
single `-i <interface>` flag, and an answer in two seconds.
For any operator-shaped LLM-CLI workflow ("the agent reports the
build is slow — is it network-bound?") it is the right
primitive: `sudo iftop -t -s 3 -L 10 -nNP -i en0` produces
deterministic plain-text output suitable for piping into the
agent's context without an X server, a browser, or a daemon.

## Vs Already Cataloged

- **Vs [`bandwhich`](../bandwhich/):** the closest peer.
  `bandwhich` is the modern Rust answer that adds *per-process*
  attribution by joining libpcap flows to `/proc/net/tcp` —
  invaluable when the question is "*which* program is using the
  bandwidth". `iftop` is older, smaller, GPL, and groups by
  `(local, remote)` host-pair without the per-process join, which
  makes it more useful when the box runs one service and the
  question is "*which remote endpoint* is the busy one". Most
  operators install both: `bandwhich` on dev laptops, `iftop` on
  servers where it's already in the distro repo and a `apt
  install` is enough.
- **Vs [`bottom`](../bottom/) / [`btop`](../btop/) /
  [`glances`](../glances/) (system monitors with a network
  panel):** those show *interface-level* totals (en0 RX/TX) in
  the corner of a multi-pane dashboard. `iftop` is the dedicated
  drill-down: per-flow rows, per-direction arrows, BPF filtering
  inside the TUI. Use the dashboards for a baseline; reach for
  `iftop` the moment the number in the dashboard's network corner
  is "weird" and you need to know which flow drove it.
- **Vs [`nethogs`](https://github.com/raboof/nethogs) (not yet
  cataloged):** `nethogs` groups by *process* (the Linux-only
  per-PID view, no host-pair table). Orthogonal axis: `iftop`
  tells you which remote you're talking to; `nethogs` tells you
  which local PID is doing the talking. Run `iftop` first to
  spot the busy flow, then `nethogs` to find the PID — the
  classic two-step.
- **Vs [`mtr`](../mtr/) / [`gping`](../gping/) /
  [`trippy`](../trippy/):** those are *path* tools (latency &
  loss across hops) for diagnosing where packets are dropped.
  `iftop` is a *volume* tool (bytes per second, here, now) for
  diagnosing who is filling the pipe. Different question, same
  network — the two together cover most "the network feels
  weird" investigations.
- **Vs [`tcpdump`](../tcpdump-not-cataloged) / [`termshark`](https://termshark.io/) / packet-level captures:** packet
  captures answer "*what bytes were on the wire*"; `iftop`
  answers "*how many bytes per second to/from each peer*". When
  you need protocol decode, capture; when you need the volume
  ranking, `iftop`. The BPF expressions transfer between the
  two so the muscle memory composes.
- **Vs interface counters via [`vnstat`](../vnstat/):** `vnstat`
  is the *historical* counter view (hour / day / month
  totals out of a tiny SQLite store, near-zero overhead because
  it polls `/sys/class/net/*/statistics` not pcap). `iftop` is
  the *live* per-flow view (libpcap capture, only while
  running). Run `vnstat` permanently for trend; reach for
  `iftop` for the drill-down.

## Caveats

- **Captures with libpcap, so it needs root / `CAP_NET_RAW`.**
  On a shared box the capture privilege is non-trivial; the
  `setcap cap_net_raw,cap_net_admin=eip $(which iftop)` trick is
  the standard mitigation. Containers need the equivalent
  capability or `--cap-add NET_RAW --cap-add NET_ADMIN` plus
  `--net=host` to see traffic outside the container's own veth.
- **Bits/sec by default, not bytes/sec.** Old-school networking
  convention; misreads as 8× too low if you instinctively read it
  as bytes. `-B` flips to bytes; pick a default and stick with it
  in your shell alias to avoid the perpetual mental ÷8.
- **DNS resolution is on by default and blocking.** Capturing on
  a busy interface with thousands of unique remote IPs with `-n`
  *off* will spam the resolver and noticeably slow the display;
  the conventional invocation on production boxes is `iftop -nNP`
  to disable both name and port resolution.
- **Per-host-pair rows, not per-process.** "Which PID owns this
  flow" is not a question `iftop` can answer — pair with
  `bandwhich` on Linux/macOS or `nethogs` on Linux for that
  attribution. Conversely, `iftop` is the only one of the three
  with a built-in BPF filter line, which often beats the others
  when narrowing to a service.
- **VLAN-tagged / encapsulated traffic counts as the outer
  packet.** `iftop` does not auto-decapsulate VXLAN / GRE /
  IPSEC; if you need the inner-flow ranking, capture upstream of
  the encap or use a tunnel-aware analyser.
- **The two upstream homes are confusing.** The original site
  (`ex-parrot.com/pdw/iftop`) is the long-time release page; the
  active source repository in 2024+ is `code.blinkace.com/pdw/iftop`
  (also Paul Warren's, a Gitea instance). Linux distro packagers
  pull from the same source tree; pin against the distro release
  unless you specifically need a newer commit. The last tagged
  release `v1.0pre4` (2018) is still the standard distro version
  precisely because the tool's surface has been stable for ~20
  years.
