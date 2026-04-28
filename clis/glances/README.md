# glances

- **Repo:** https://github.com/nicolargo/glances
- **Version:** v4.5.4 (latest stable release, 2026-04)
- **License:** LGPL-3.0 ([COPYING](https://github.com/nicolargo/glances/blob/develop/COPYING))
- **Language:** Python (3.10+)
- **Install:** `pipx install 'glances[all]'` · `uvx glances` · `brew install glances` · `pip install glances` · Docker `nicolargo/glances:latest-full`

## What it does

`glances` is a cross-platform system monitor written in Python that
takes the `top` / `htop` / `vmstat` / `iostat` / `iftop` / `nethogs`
swarm and rolls them into one curses dashboard with per-plugin colour-
coded thresholds. The core view shows CPU (per-core, with iowait /
steal / softirq broken out), load average, memory and swap with
buffer/cache breakdown, disk I/O per device, network throughput per
interface, file-system usage, sensors (temperature, fan, battery), the
process table sorted by any column you like, and on Linux: containers
(Docker / Podman / LXC), GPU (NVIDIA), RAID, SMART. Where glances
diverges from `htop` is the integration surface: every metric the TUI
shows is also exposed via a built-in REST API (`glances -w` boots a
FastAPI web UI on port 61208 with an OpenAPI schema), via XML-RPC
client/server (`glances -s` on the box, `glances -c <ip>` from your
laptop), via stdout in CSV / JSON / templated formats for piping into
scripts, and via direct export to InfluxDB / Prometheus / Graphite /
Elasticsearch / Kafka / StatsD / Cassandra / ClickHouse / OpenTSDB /
RabbitMQ / NATS / ZeroMQ — i.e. it ships as a one-binary metrics
collector for a small fleet without forcing you to install a real
agent like Telegraf. The 4.5.x line also added an MCP server
(`--enable-mcp`) so an LLM agent can query the live machine state over
SSE. Configuration is a single ini file with per-plugin enable / disable
/ threshold blocks, and a `--browser` mode auto-discovers other
glances servers on your LAN via Zeroconf.

## When to pick it / when not to

Pick `glances` when you want a "single tool" answer to "what's
happening on this box right now" that goes beyond what `htop` shows —
you get sensors, network, disks, containers, GPU, and the process
table on one screen, with thresholds already tuned. Pick it
specifically when you need a *remote* monitor without standing up
Prometheus + node_exporter + Grafana: `glances -w` on the server +
your browser is a five-minute deployment that is good enough to debug
"why is this VM slow" from another continent. Pick it when you're
building an LLM agent that should be able to ask "what's the CPU
load on host X" — the REST API and the new MCP server give you that
directly. The Python implementation is also the right call when you
already have the Python runtime there (cloud VMs, dev containers,
Raspberry Pis) and don't want a Go / Rust binary to manage.

Skip it when you want the absolute lowest-overhead per-second TUI on a
constrained box — [`btop`](../btop/) and [`bottom`](../bottom/) are
native binaries (C++ / Rust) with prettier graphs, lower CPU cost, and
no Python interpreter sitting in your `ps` output. Skip it when you
need *real* time-series monitoring (alerting, multi-host dashboards,
30-day retention) — that's Prometheus + Grafana / VictoriaMetrics
territory, and exporting to InfluxDB from glances is fine for a small
homelab but won't scale to a fleet. Skip it for process-tree
exploration where [`htop`](https://htop.dev) F5 / `procs --tree` are
much better. Skip it on platforms where `psutil` cannot install
cleanly (rare, but happens on exotic embedded targets) — the `top`
binary is always there and you already know how to read it. Note also
that "all features" (`pip install 'glances[all]'`) pulls in a *lot* of
optional deps; for a bare TUI on a server, the plain `pip install
glances` install is much smaller.

## Example invocations

```bash
# Standalone curses TUI, refreshes every second
glances

# Web UI on port 61208 (works as a remote dashboard)
glances -w

# Web UI plus MCP server for LLM agents (4.5.1+)
glances -w --enable-mcp

# Server side: expose XML-RPC stats
glances -s

# Client side: connect to a remote glances server
glances -c 10.0.0.42

# Pipe a few specific metrics to stdout (for cron / scripts)
glances --stdout cpu.user,mem.used,load

# Same, but CSV format
glances --stdout-csv now,cpu.user,mem.used,load

# Discover other glances servers on the LAN (Zeroconf)
glances --browser

# Quick fetch — print one snapshot and exit (good for SSH one-liners)
glances --fetch

# Push metrics to InfluxDB v2
glances --export influxdb2

# Long-running on a server, exporting to Prometheus, no UI
glances --quiet --export prometheus
```
