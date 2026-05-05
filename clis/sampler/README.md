# sampler

> **Shell-command dashboards from one YAML file** — a Go binary
> that turns periodic shell invocations into live charts, gauges,
> sparklines, runcharts, bar plots, asciiboxes, and textboxes,
> with optional per-cell triggers that fire on threshold crossings.
> Pinned to **v1.1.0** (commit
> `43fa3049dba1c8bafc5592c7427f79489a527f3b`,
> [LICENSE.md](https://github.com/sqshq/sampler/blob/master/LICENSE.md),
> GPL-3.0).

Source: <https://github.com/sqshq/sampler>

## TL;DR

`sampler` reads a single YAML file describing a grid of "samples"
— each sample is a shell command, a polling rate, and a
visualisation type. On each tick the command runs, its stdout is
parsed (numeric for charts, text for boxes), and the result feeds
the cell's renderer. The grid is drawn full-screen with a
ratatui-style TUI; cells can carry triggers (`condition: ...; actions:
[say: ..., trigger-sound: true, terminal-bell: true, script: ...]`)
that execute when their boolean expression flips. The output is
the kind of bespoke dashboard that would otherwise require
Grafana + Prometheus + an exporter, but the data source is
"whatever shell command produces a number".

## Install

```bash
# Homebrew (macOS / Linux)
brew install sampler

# Pre-built binary (Linux)
curl -Lo sampler https://github.com/sqshq/sampler/releases/download/v1.1.0/sampler-1.1.0-linux-amd64
chmod +x sampler && sudo mv sampler /usr/local/bin/

# from source (requires Go + ALSA dev headers on Linux for sound triggers)
go install github.com/sqshq/sampler@v1.1.0

# verify
sampler --version    # 1.1.0

# run with an explicit config
sampler --config ./dashboard.yml
```

Configuration is one YAML file (path passed via `-c`, no implicit
`~/.config/sampler.yml`). Samples are arranged on a grid by
`(position: [x, y, width, height])`; the renderer types are
`runcharts`, `sparklines`, `barcharts`, `gauges`, `asciiboxes`,
`textboxes`, and `searchboxes`.

## License

GPL-3.0 — see
[LICENSE.md](https://github.com/sqshq/sampler/blob/master/LICENSE.md).
Distribute the source for any modifications you redistribute as
binaries; safe to use as a personal / internal-team tool without
relicensing your monitored systems.

## One Concrete Example

```yaml
# dashboard.yml — disk + memory + service-health dashboard
runcharts:
  - title: CPU usage by core
    rate-ms: 500
    scale: 2
    items:
      - label: CPU1
        sample: ps -A -o %cpu | awk '{s+=$1} END {print s/4}'
      - label: load-1m
        sample: uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | tr -d ','

gauges:
  - title: Disk usage /
    rate-ms: 5000
    cur:
      sample: df / | tail -1 | awk '{print $5}' | tr -d '%'
    max:
      sample: echo 100
    min:
      sample: echo 0

sparklines:
  - title: Active TCP connections
    rate-ms: 1000
    sample: netstat -an | grep ESTABLISHED | wc -l

textboxes:
  - title: Top 5 processes by RSS
    rate-ms: 2000
    sample: ps -A -o %mem,comm | sort -rn | head -5

barcharts:
  - title: Postgres connections per database
    rate-ms: 5000
    items:
      - label: prod
        sample: psql -At -U monitor -c "SELECT count(*) FROM pg_stat_activity WHERE datname='prod'"
      - label: stage
        sample: psql -At -U monitor -c "SELECT count(*) FROM pg_stat_activity WHERE datname='stage'"

triggers:
  - title: Disk-full alarm
    condition: '[ $(df / | tail -1 | awk "{print \$5}" | tr -d %) -gt 85 ]'
    actions:
      terminal-bell: true
      sound: true
      visual: true
      script: notify-send "Disk > 85%"
```

```bash
sampler -c dashboard.yml
```

## Niche It Fills

**Bespoke shell-driven dashboards without a metrics stack.** The
typical "I want a live view of these eight numbers from this
host" workflow is either three `tmux` panes running `watch`, or
standing up Prometheus + node-exporter + Grafana for what amounts
to one screen. `sampler` is the in-between: declare the grid + the
shell commands + the refresh rates in one YAML, run one binary,
get the dashboard. No agent, no time-series DB, no browser tab.
The data source is "what `ssh` and `awk` can produce", which is
broader than any exporter library.

## Why use it

1. **Shell command is the data source.** Anything you can pipe
   into `awk '{print $1}'` is a sample — `psql -At`, `redis-cli`,
   `aws cloudwatch get-metric-statistics`, `curl /metrics | grep
   foo`, `kubectl get pods | wc -l`, `ipmitool sensor | awk`. No
   exporter to write, no scrape config, no client library
   linkage. The cost of monitoring a new metric is one YAML
   stanza.
2. **Triggers carry actions, not just thresholds.** A condition
   that flips can play a sound, ring the terminal bell, run a
   `notify-send` / `osascript` / `tmux display-message`, or shell
   out to an arbitrary script. The dashboard is also a poor-man's
   Alertmanager for the cases where a 30-second feedback loop is
   the whole point.
3. **YAML in version control.** A team's dashboards live as
   `dashboards/*.yml` in the infra repo, reviewed in PRs the
   same way runbooks are. No JSON dump from a Grafana export, no
   click-to-recreate-on-fresh-install. New laptop, `sampler -c
   prod.yml`, full dashboard.

For an LLM-CLI workflow, `sampler` is rarely a tool the agent
calls directly; it's the *operator's* observation surface for
"what is the agent doing right now" (queue depth, model latency,
token-spend per minute) when the agent's own logs don't aggregate.

## Vs Already Cataloged

- **Vs [`gtop`](../gtop/) / [`bottom`](../bottom/) / [`btop`](../btop/) / [`glances`](../glances/):**
  those are *fixed* system-monitor TUIs (CPU, memory, network,
  disk are hardcoded panels). `sampler` is a *configurable* grid
  where the panels are whatever you wrote in the YAML — including
  panels the system monitors will never have (Postgres connections
  per database, queue depth in your job system, the API
  rate-limit headroom). Compose: leave the system monitor running
  in pane A, run `sampler` with your domain-specific dashboard in
  pane B.
- **Vs [`viddy`](../viddy/) / [`hwatch`](../hwatch/):** those are
  better `watch(1)` — one shell command, one pane, time-travel
  diff of its output. `sampler` is multi-cell with native chart
  renderers. Pick `viddy`/`hwatch` for "watch this one command
  and diff its output"; pick `sampler` for "draw eight metrics on
  one screen with charts".
- **Vs [`grafana-cli`](../grafana-cli/) (not cataloged) / Grafana
  + Prometheus:** sampler is the right answer when the metric
  set is small, the audience is one terminal, and standing up a
  full metrics stack is overkill. Grafana wins above ~50 metrics,
  retention beyond memory, multi-user access, and the alerting +
  RBAC story.
- **Vs [`vector`](../vector/) / [`fluent-bit`](../fluent-bit/) /
  metric agents:** orthogonal — those *collect and forward*
  metrics to a backend; sampler *renders* current values from
  shell commands directly. They live at different layers.

## Caveats

- **Project velocity is low.** v1.1.0 was tagged in 2021; the
  repo is in maintenance mode (last push 2024). It still works
  on current Go / macOS / Linux but expect no new renderers.
  Pin the version in your install script and audit before
  upgrading.
- **One YAML is the entire trust boundary.** Triggers can run
  arbitrary shell commands; treat shared dashboards the same way
  you treat shared `Makefile`s. Don't `sampler -c
  https://...whatever` against an untrusted URL.
- **Per-cell `rate-ms` runs the command in a separate fork each
  tick.** A 200 ms sparkline polling `psql -At` against a remote
  Postgres at 5/s adds load — sampler does not connection-pool
  for you. Bump the polling rate to a value the data source can
  serve.
- **GPL-3.0 affects redistribution, not internal use.** Running
  sampler on your fleet is fine; redistributing a fork as
  closed-source binaries is not. Most "team dashboard" use
  never approaches the boundary.
- **Sound triggers need ALSA / CoreAudio at runtime.** On
  headless servers and minimal containers the sound action
  no-ops silently. Use `terminal-bell` + `script: ...` for
  always-available notification paths.
- **Numeric parsing is `strconv.ParseFloat` on stdout.** A
  command that prints `"42 OK"` will fail to parse; pipe through
  `awk '{print $1}'` to ensure the cell receives a bare number.
