# vnstat

- **Upstream:** https://github.com/vergoh/vnstat
- **Version:** v2.13 (released 2025-02-08)
- **License:** GPL-2.0 (`COPYING`)
- **Language:** C

## What it is

`vnstat` is a lightweight network-traffic monitor that records per-interface RX / TX byte
counters into a small on-disk database (default `/var/lib/vnstat/`) on a 5-minute cadence and
exposes the rollups (5min / hourly / daily / monthly / yearly / top-days) through a CLI. Unlike
`iftop` / `nethogs` / `bandwhich` (which sniff packets and need root + a live capture), `vnstat`
just polls the kernel's interface counters (`/proc/net/dev` on Linux, `getifaddrs` on BSD /
macOS), so it is essentially free in terms of CPU and works correctly on a low-power
device or a remote VPS over an SSH session. The companion `vnstati` binary renders the same
data as PNG charts for a status page or webhook attachment, and `vnstatd` is the optional daemon
that keeps the database current when invoked under systemd / launchd / cron.

## Why an AI/CLI user might pick it

For "how much data did this box actually move last month?" — the question every cloud bill,
every metered home connection, and every "is that training run hitting the egress cap?"
investigation eventually boils down to — `vnstat` is the right shape: persistent, low-overhead,
queryable from a script (`vnstat --json m` emits month rollups as JSON ready for `jq`), and
coexists trivially with everything else on the machine. v2.13 added `--db` to query an arbitrary
database file, exit-status differentiation in `--alert` mode (so cron can distinguish "threshold
crossed" from "vnstat itself errored"), and a merge mode for combining databases when migrating
hosts. Pairs with `bandwhich` (real-time per-process attribution) and `iperf3` (active throughput
tests); `vnstat` is the long-horizon counter the other two cannot give you.

## Install

```sh
brew install vnstat
```

## Example

```sh
# Initialize the database for an interface and print the monthly summary as JSON
vnstat --add -i en0
vnstat -i en0 --json m | jq '.interfaces[0].traffic.month[-1]'
```
