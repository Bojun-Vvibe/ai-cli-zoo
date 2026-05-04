# nethogs

> **Per-process network bandwidth monitor** — a small C++
> ncurses TUI that groups live network throughput **by
> process** (not by host, port, or interface) by reading
> `/proc/net/tcp` + `/proc/net/tcp6` + `/proc/net/udp` and
> matching socket inodes back to PIDs via `/proc/<pid>/fd/`,
> then sniffing packet sizes off the wire with `libpcap` and
> attributing each packet to the owning process — pinned to
> **v0.8.8** (commit
> [`632a7884`](https://github.com/raboof/nethogs/commit/632a78846eb3cc3259dc45c59a47fa9c293a2831),
> [COPYING](https://github.com/raboof/nethogs/blob/v0.8.8/COPYING),
> GPL-2.0-or-later).

Source: <https://github.com/raboof/nethogs>

## TL;DR

`top` for network bandwidth, keyed by PID. The screen is one
row per process actively sending or receiving on the network,
sorted by current throughput, with `SENT` / `RECEIVED` columns
in KB/s and a running total per row. When the laptop fan
spins up because something on the box is uploading at
8 MB/s, `sudo nethogs` answers **which process** in one
keystroke — without parsing `ss`, correlating `lsof -i`
output to `iftop`'s per-flow view, or guessing from
`/proc/net/dev` interface counters that aggregate all
processes together.

The killer property is **PID-keyed attribution on a shared
host**. `iftop` shows per-flow (host:port ↔ host:port)
bandwidth, `bandwhich` shows per-connection + per-process on
modern Linux/macOS, but on a multi-tenant Linux box (CI
runner, build server, shared dev VM) where 50 processes are
all talking to the same registry mirror, nethogs's
**process-rollup view** is the one that surfaces "the
runaway is `pid 18432 / curl`" instead of "there are
80 flows to 10.0.0.5:443."

## Install

```bash
# Debian / Ubuntu
sudo apt install nethogs

# Fedora
sudo dnf install nethogs

# Arch Linux (extra)
sudo pacman -S nethogs

# Alpine
sudo apk add nethogs

# Build from source
git clone --branch v0.8.8 https://github.com/raboof/nethogs
cd nethogs
make && sudo make install

# verify
nethogs -V    # version 0.8.8
```

Requires `libpcap` (runtime) and `CAP_NET_ADMIN` +
`CAP_NET_RAW` to sniff packets — invoke via `sudo nethogs`,
or grant the binary capabilities once with
`sudo setcap 'cap_net_admin,cap_net_raw+pe' $(which nethogs)`
and run unprivileged thereafter.

## Example usage

```bash
# default: all interfaces, interactive ncurses TUI
sudo nethogs

# specific interface (laptop wifi)
sudo nethogs wlan0

# multiple interfaces at once
sudo nethogs eth0 wlan0

# tracemode: line-per-update on stdout (no ncurses, pipe-able)
sudo nethogs -t                       # KB/s columns
sudo nethogs -t -d 5                  # update every 5 s

# show in KB/s totals only (no per-row averages)
sudo nethogs -v 1                     # 0=KB/s, 1=total KB, 2=total B, 3=total MB

# include the destination IP / port in the row label
sudo nethogs -C                       # capture mode shows connection details

# headless / scriptable: 10 samples then exit
sudo nethogs -t -c 10 > nethogs.log

# in-TUI hotkeys
#   m  cycle units (KB/s → KB total → MB total → B total)
#   r  sort by RECEIVED
#   s  sort by SENT
#   q  quit
```

Common flags:

- `-d SECS` refresh interval (default 1 s)
- `-v MODE` units: `0` KB/s, `1` total KB, `2` total B, `3` total MB
- `-c COUNT` exit after N updates (scriptable runs)
- `-t` tracemode — plain stdout, no ncurses (pipe to `awk`/`jq`)
- `-b` bughunt mode (extra debug output)
- `-C` capture mode — show per-connection rows (PID + endpoints)
- `-p` promiscuous mode (capture traffic not destined for this host)
- `-s` sort by sent / `-r` by received (TUI hotkeys also)
- `-V` / `-h` version / help

## Why it matters

- **PID is the question, not flow.** When a host's outbound
  bandwidth is saturated, the operator question is "which
  process" — the answer that lets you `kill -STOP` or `nice`
  the offender. nethogs answers it directly; `iftop` /
  `tcpdump` / `ss -tulpn` need additional join steps to get
  there.
- **Works on headless servers / containers.** No GTK / Qt /
  D-Bus / X dependency — just ncurses + libpcap. Drops cleanly
  into a `tmux` pane on a remote box where graphical
  monitoring (Grafana, Prometheus node_exporter dashboards)
  is overkill or not yet wired up.
- **Tracemode is `awk`-friendly.** `nethogs -t -d 5` emits
  one block of text per refresh; pipe through `awk`/`jq` for
  ad-hoc alerting (e.g. "page me if any single PID exceeds
  20 MB/s for 30 s") without standing up a metrics pipeline.
- **Capability-based unprivileged operation.** Set
  `cap_net_admin,cap_net_raw+pe` once and developers can
  run nethogs without `sudo` — the standard hardened-server
  pattern, distinct from giving them broad root.
- **Multi-interface aggregation.** `sudo nethogs eth0 wlan0`
  on a dual-homed laptop / router shows one unified
  per-process view across both NICs, which `iftop` (one
  interface per invocation) requires two side-by-side panes
  to replicate.

## Vs Already Cataloged

- ai-cli-zoo does not currently catalog any per-process
  network monitor; nethogs is the first entry in this niche.
  It composes with [`procs`](../procs/) (per-process *CPU /
  memory* in a modern `ps`-replacement table) — procs answers
  "which process is hot on CPU," nethogs answers "which
  process is hot on the network." Both share the
  PID-as-primary-key model.
- **Vs `iftop` / `iptraf-ng`:** orthogonal — `iftop` keys on
  flows (host:port ↔ host:port), `iptraf-ng` on
  protocols / interfaces. Use those when the question is
  "which remote endpoint is the hot one"; use nethogs when
  the question is "which local process is the hot one."
- **Vs `bandwhich`:** overlapping but different surfaces —
  `bandwhich` is a Rust TUI with per-connection AND
  per-process rows, supports macOS/BSD via `pcap`, and emits
  raw stats. nethogs is older, C++, Linux-first, and the
  PID-rollup view is its default — when the operator wants
  one row per process and nothing else on the screen, nethogs
  is the cleaner fit. (Mentioned here for context;
  `bandwhich` is not in the zoo at time of writing.)
- **Vs `ss -tulpn` / `lsof -i`:** snapshot tools — they list
  *which* sockets a process holds, not *how much traffic*
  flows through them. Compose: `ss` to confirm the
  connection exists, nethogs to see if it is moving bytes.
- **Vs `tcpdump` / `wireshark`:** packet-level inspection —
  the right tool for "what is in the bytes." nethogs answers
  the prior question of "which process is sending so many
  bytes that I want to inspect them." Often run together.
- **Vs Grafana + node_exporter / netdata / Prometheus:** the
  long-running observability stack records bandwidth over
  hours/days and supports alerting. nethogs is the
  ten-second answer when the dashboard is not yet wired up
  or a problem is happening *right now* on a host without
  metrics export.

## License

GPL-2.0-or-later — see
[COPYING](https://github.com/raboof/nethogs/blob/v0.8.8/COPYING).
nethogs is a standalone binary; running it from any script
or pipeline is unrestricted. Static linking against
`libnethogs` (the sibling library) into a non-GPL-compatible
binary is restricted by the GPL — use the CLI's tracemode
output as an interop layer instead.

## Caveats

- **Linux-only.** nethogs depends on `/proc/net/*` socket
  tables and `/proc/<pid>/fd/` inode mapping — Linux
  kernel surfaces. macOS / BSD users should use
  `bandwhich` (cross-platform pcap-based per-process
  attribution) or platform-native tools (`nettop` on
  macOS).
- **Requires elevated privileges.** Packet capture needs
  `CAP_NET_RAW` + `CAP_NET_ADMIN`; default install runs
  via `sudo`. The `setcap` workaround grants the binary
  capabilities permanently — review your threat model
  before doing this on a multi-user host.
- **Sniffer-based attribution misses kernel-internal
  traffic.** Anything that does not traverse the
  packet-capture path (e.g. some VPN tunnels at certain
  layers, kernel-internal NFS RDMA) will not be visible.
  Standard userland TCP/UDP traffic — the 99% case — is
  fine.
- **PID matching has a small race window.** A short-lived
  process that opens a socket, sends a burst, and exits in
  under one refresh tick may appear as "unknown" because
  `/proc/<pid>/fd/` is gone by the time nethogs correlates.
  Lower `-d` to `1` or `0.5` if hunting short bursts.
- **No per-container view by default.** Containers share
  the host's PID namespace from nethogs's vantage point
  (it sees the host PIDs of containerized processes).
  For per-container rollup, run nethogs *inside* each
  container's net namespace (`nsenter -n -t <pid>
  nethogs`) or use a container-aware tool like `cadvisor`.
- **Last upstream release was 2023-09-09 (v0.8.8).** nethogs
  is "stable not stalled" — the `/proc` + `libpcap` surfaces
  it depends on are ABI-stable Linux kernel interfaces, and
  the daily-driver workflows have not needed updates.
  Active issue tracker; monitor for kernel-API breakage on
  far-future kernels.

## As of

2026-05-04. Upstream tag `v0.8.8` (2023-09-09). The Linux
`/proc/net` and `libpcap` surfaces nethogs depends on are
stable; re-verify only if a future v0.9 or v1.0 release
reshapes the CLI flags or output format.
