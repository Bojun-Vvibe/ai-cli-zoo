# ttyplot

- **Repo:** https://github.com/tenox7/ttyplot
- **Version:** 1.7.1 (released 2025-03-19)
- **License:** Apache-2.0 ([LICENSE](https://github.com/tenox7/ttyplot/blob/master/LICENSE))
- **Language:** C
- **Install:** `brew install ttyplot` · `apt install ttyplot` · `pacman -S ttyplot` · `pkg install ttyplot` · build with `make` (single C file, only `ncurses` required) · binary name is `ttyplot`

## What it does

`ttyplot` is a **realtime line plotter for the terminal**. It reads
numeric values from stdin one per line (or two per line for
two-series plots) and continuously redraws an `ncurses` line chart
that scrolls left as new samples arrive. It auto-scales the Y axis
to the current data range, prints the latest / min / max / mean in
the corner, supports two-series overlay plots (good for "in vs out"
or "before vs after" pairs), an optional rate mode (`-r`) that
treats the input as a monotonically increasing counter and plots the
per-second delta, a units suffix (`-u Mbps`), a custom title
(`-t "swap pressure"`), and a hard Y-max (`-s 100`) to lock the
axis when comparing across runs.

## When to pick it / when not to

Pick `ttyplot` when you have **a stream of numbers and you want to
see the shape of it without leaving the terminal**: `ping -i 0.2
8.8.8.8 | sed -u 's/.*time=\([0-9.]*\).*/\1/' | ttyplot -t "RTT"`
shows packet RTT live; `vmstat 1 | awk 'NR>2 {print $13; fflush()}'
| ttyplot -t "user CPU%"` plots `vmstat`'s user-CPU column; an
arbitrary script that writes `printf '%s\n' "$value"` in a loop
becomes graphable for free. The killer property is that it composes
through pipes — you don't write a config, you don't open a UI, you
don't pick a metric name; you just pipe a number stream in.

Skip it when the data is **multi-dimensional or labelled** —
[`youplot`](../youplot/) handles bar / scatter / histogram / heatmap
shapes and is the right reach for one-shot statistical plots over a
column. Skip it for **system-monitoring dashboards with built-in
metric collection** ([`bottom`](../bottom/), [`btop`](../btop/),
[`gtop`](../gtop/), [`zenith`](../zenith/), [`s-tui`](../s-tui/)) —
those collect their own metrics and lay out a dashboard; `ttyplot`
is the dual: BYO-metric, single-series, one panel.

## Why it belongs in the zoo

The zoo has a deep bench of *system-monitor* TUIs that own both the
data collection and the rendering ([`bottom`](../bottom/),
[`btop`](../btop/), [`gtop`](../gtop/), [`zenith`](../zenith/),
[`bandwhich`](../bandwhich/)) and a *statistical-plot* one-shot
[`youplot`](../youplot/) for offline column data. `ttyplot` sits in
the orthogonal slot: a **streaming plotter that does no collection
of its own**, so any shell pipeline that emits a number per line
becomes a live chart. Nothing else in the catalog occupies that
"unix-pipe in, ncurses chart out" niche. Apache-2.0 makes it safe to
vendor into commercial monitoring runbooks.

## Example invocations

```bash
# Live ping RTT to a host, one sample every 200 ms
ping -i 0.2 8.8.8.8 \
  | sed -u 's/.*time=\([0-9.]*\).*/\1/' \
  | ttyplot -t "8.8.8.8 RTT (ms)"

# Two-series: bytes-in vs bytes-out from a custom counter exporter
while sleep 1; do
  read rx tx < <(awk '/^ *eth0:/ {print $2, $10}' /proc/net/dev)
  printf '%s %s\n' "$rx" "$tx"
done | ttyplot -2 -r -u "B/s" -t "eth0 rx (-) vs tx (--)"

# Free memory in MB sampled once per second, fixed Y-max for stability
vmstat 1 | awk 'NR>2 {print $4/1024; fflush()}' \
  | ttyplot -s 16384 -u MB -t "free RAM (MB)"

# Plot a custom application metric streamed over SSH
ssh prod-1 'tail -F /var/log/myapp/queue-depth.log' | ttyplot -t "queue depth"
```
