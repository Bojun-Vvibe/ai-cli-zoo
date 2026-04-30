# below

- **Repo:** https://github.com/facebookincubator/below
- **Latest version:** v0.11.0 (2025-09-24)
- **HEAD on `main`:** `b54a600`
- **License:** Apache-2.0 — [LICENSE](https://github.com/facebookincubator/below/blob/main/LICENSE)
- **Category:** sysadmin / system monitor

A **time-traveling resource monitor** for modern Linux systems. Unlike
`top` / `htop` / [`bottom`](../bottom/), `below` runs as a tiny daemon
that records cgroup-v2-aware system telemetry to a local on-disk store,
so you can scroll *backwards* through CPU, memory, IO, network, and
pressure-stall (PSI) metrics minutes or hours after the incident has
passed. The TUI lets you replay any window with arrow-key navigation,
group by cgroup / process / iface / disk, and dump any view to CSV /
JSON for postmortems. Built by Meta's production-engineering team and
driven by their incident-review workflow — the design priority is
"answer 'what happened at 03:42?' on a host that was paged at 04:00".

## Install

```bash
# Linux only — depends on cgroup-v2, /proc, /sys, BPF where available

# Fedora / RHEL / CentOS Stream
dnf install below

# Arch (AUR)
yay -S below

# from source (Rust toolchain required)
cargo install --locked below

# Debian / Ubuntu: build from source or grab a release tarball
curl -LO https://github.com/facebookincubator/below/releases/download/v0.11.0/below-v0.11.0-x86_64-unknown-linux-gnu.tar.gz

# enable the recorder daemon (writes to /var/log/below by default)
systemctl enable --now below

# verify
below --version    # below 0.11.0
```

## Sample usage

```bash
# live view (top-style, cgroup-aware)
below live

# replay the last 30 minutes
below replay --time "30m ago"

# jump to a specific timestamp
below replay --time "2026-04-29 14:32:00"

# dump per-cgroup CPU + memory for the last hour, as CSV
below dump cgroup --begin "1h ago" --end now --output-format csv \
  --fields name cpu_usage_pct mem_total > cgroups.csv

# dump per-process IO for postmortem
below dump process --begin "2026-04-29 14:00" --end "2026-04-29 14:10" \
  --output-format json --fields pid comm io_read_bytes io_write_bytes

# show pressure-stall (PSI) for memory across all cgroups
below replay --time "1h ago"   # then press 'p' for pressure view
```

## Niche it fills

The **"recorder + replayer for cgroup-v2 telemetry"** slot. Where
[`bottom`](../bottom/), `htop`, and `glances` show *now* and forget
the past, `below` is built around the assumption that you will be asked
about an incident after it ended. Where Prometheus + Grafana solve the
same problem at fleet scale (with significant operational cost), `below`
solves it on a single host with one daemon, one binary, and a flat-file
store — no Prometheus, no exporter, no dashboarding stack.

## Why it matters

- **Native cgroup-v2 model.** Process trees are grouped the way systemd
  / Kubernetes / nspawn actually slice the system, not by ad-hoc PPID
  walking. Memory pressure on a single pod is a row, not a deduction.
- **PSI as a first-class column.** Linux pressure-stall information
  (`/proc/pressure/{cpu,memory,io}`) is the single best signal for
  "this box is contended" and most monitors still ignore it; `below`
  surfaces it in the live view and the replay.
- **Time travel without a TSDB.** Replays are local-disk only, no
  network round-trip, default retention is days. Good for "what was
  happening when the OOM killer fired at 03:42?" without standing up
  Prometheus.
- **Dump subcommand is the scripting contract.** `below dump` emits
  CSV / JSON / raw and is the right tool to pipe historical telemetry
  into a notebook or `jq` for an after-the-fact analysis.

## Caveats

- **Linux only, cgroup-v2 only.** No macOS, no FreeBSD, no Windows; on
  Linux distros still booting cgroup-v1 you get a degraded view.
- **The recorder is a daemon and uses disk.** Default retention writes
  hundreds of MB / day on a busy host; tune
  `/etc/below/below.conf:store_size_limit` for small VMs.
- **TUI is keyboard-only and dense.** No mouse, no themes; the learning
  curve is roughly an evening of `?` to memorise the bindings.
- **Not a fleet tool.** Each host has its own store; for cross-host
  views you still want Prometheus / OpenTelemetry. `below` is the
  per-host complement, not a replacement.
