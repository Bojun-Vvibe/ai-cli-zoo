# rare

> **Realtime regex-driven log analysis & terminal
> visualisation** — point it at one or more (possibly
> compressed, possibly tailed) log files, give it a regex with
> capture groups, and it streams the matches into live
> histograms, bar charts, sparklines, tables, heatmaps, or
> time-bucketed counters at multi-GB/s throughput, with a
> small expression language for math, date parsing, and
> grouping. Pinned to **0.5.6** (released 2026-02-08,
> [LICENSE](https://github.com/zix99/rare/blob/0.5.6/LICENSE),
> GPL-3.0).

Source: <https://github.com/zix99/rare>

## TL;DR

The shape "tail a log, group by some field, watch counts
update live" usually means a five-tool pipeline:
`tail -f | grep | awk '{print $7}' | sort | uniq -c | sort -rn`,
which is non-incremental, blocks on `sort`, and tells you
nothing about distributions or time. `rare` collapses the
whole pipeline into one command: a single regex pulls capture
groups out of the stream, an aggregator (`histo`, `bars`,
`heatmap`, `tabulate`, `analyze`, `filter`, `reduce`)
maintains running state in memory, and the Bubble-Tea-style
TUI repaints the chart in place every 100 ms — no waiting
for end-of-file, no shelling out to gnuplot. Single static Go
binary, ~5.5 GB/s peak on a modern NVMe.

## Install

```bash
# macOS (Homebrew tap)
brew install zix99/tap/rare

# Linux (Arch via AUR, deb / rpm / apk on releases page)
yay -S rare
# or:
# https://github.com/zix99/rare/releases/tag/0.5.6
sudo dpkg -i rare_amd64.deb

# Go install (cross-platform)
go install github.com/zix99/rare@0.5.6

# Install script
curl -sSL https://raw.githubusercontent.com/zix99/rare/0.5.6/install.sh | bash

# Verify
rare --version            # rare 0.5.6
```

## License

GPL-3.0 — see
[LICENSE](https://github.com/zix99/rare/blob/0.5.6/LICENSE).
Copyleft: redistributing a modified `rare` binary requires
the same license; running `rare` against your own log files
and shipping its *output* is unencumbered (calling a GPL
binary from a proprietary script does not infect the script).

## Common invocations

```bash
# Live histogram of HTTP status codes from an nginx log
rare histo -m '"(\S+) (\S+) HTTP[^"]*" (\d+)' -e '{3}' /var/log/nginx/access.log

# Bar chart of top 20 request paths, updates as the file grows
rare bars -m '"(\S+) (\S+) HTTP' -e '{2}' -n 20 -f /var/log/nginx/access.log

# Time-bucketed heatmap: status code by minute over the last hour
rare heatmap \
  -m '\[(\S+ \S+)\].*HTTP[^"]*" (\d+)' \
  -e '{bucket {time {1} "02/Jan/2006:15:04:05"} "1m"}' \
  -e '{2}' \
  /var/log/nginx/access.log

# Tabulate: bytes-out per user-agent class, sorted
rare tabulate \
  -m '"(\S+) HTTP[^"]*" \d+ (\d+) "[^"]*" "([^"]*)"' \
  -e '{3}' -e '{sumi {2}}' \
  -s avg /var/log/nginx/access.log

# Filter: stream only lines where extracted latency > 500ms
rare filter -m 'latency=(\d+)ms' -e '{gt {1} 500}' app.log

# Walk a directory tree (since 0.5.0) and count error lines per file
rare bars --walk ./logs -m '\bERROR\b' -e '{src}'

# Read a gzip / bz2 / xz / zstd archive directly, no piping
rare histo -m 'level=(\w+)' -e '{1}' app.log.zst

# Pipe stdin from another tool
journalctl -f | rare bars -m 'unit=(\S+)' -e '{1}'
```

## Why use it

- **One regex + one expression replaces a five-tool pipe.**
  Capture groups feed an expression language (`{eq}`,
  `{gt}`, `{sumi}`, `{bucket}`, `{time}`, `{coalesce}`,
  `{json}`, `{multiply}`, `{divide}`, `{percent}`, etc.)
  that does the math, date parsing, and bucketing inline —
  no `awk`, no `date -d`, no `bc`.
- **Truly incremental.** Aggregators maintain running state
  per second; you watch the chart take shape live, no
  end-of-file required. `-f` follows like `tail -f`.
- **Multi-format input out of the box.** Plain text, gzip,
  bzip2, xz, zstd, and stdin all decode transparently.
  `--walk DIR` (added in 0.5.0) recurses a tree with
  `--include` / `--exclude` glob filters and parallel
  workers.
- **Fast.** Goroutine reader pool + buffered extractor +
  pre-typed expression stages reach ~5.5 GB/s on the
  author's NVMe benchmark (0.5.2). For most practical log
  sizes you are I/O-bound, not CPU-bound.
- **PCRE optional.** Default regex is Go's RE2 (linear time,
  safe). `--pcre` switches to a PCRE2 engine when you need
  lookbehind / backrefs that RE2 refuses.

## Vs Already Cataloged

- **Vs [`angle-grinder`](../angle-grinder/) / `agrind`:**
  closest neighbour. `agrind` ships a SQL-ish pipeline
  language (`* | parse "..." | sum bytes by host`); `rare`
  ships regex + a small Lisp-like expression DSL plus
  built-in chart aggregators (`bars`, `histo`, `heatmap`).
  `agrind` wins on JOIN-shaped questions; `rare` wins on
  "watch this distribution change in real time" with native
  visualisations.
- **Vs [`lnav`](../lnav/):** `lnav` is the full curses log
  navigator with a SQL-over-virtual-tables engine and an
  annotation UI; multi-file timeline reading is its
  strength. `rare` is non-interactive — one regex, one
  chart, one shell command, repaint forever — closer to a
  Unix filter than a navigator.
- **Vs [`fblog`](../fblog/) / [`tspin`](../tspin/) /
  [`tailspin`](../tailspin/):** those are colourisers /
  pretty-printers / structured-log filters. `rare`
  *aggregates* — its output is counters and charts, not a
  prettier copy of the input lines.
- **Vs [`miller`](../miller/) / [`mlr`](../mlr/):** `mlr`
  is a record-oriented DSL for CSV / JSONL / TSV with stats
  built in (`stats1`, `stats2`, `histogram`); excellent for
  finite, structured datasets. `rare` is regex-on-text and
  optimised for streaming `tail -f`-style live updates.
- **Vs [`humanlog`](../humanlog/) / [`logdy`](../logdy/):**
  log readers/colourisers with web UIs. Different
  category — they help humans read; `rare` aggregates.
- **Vs `goaccess`:** `goaccess` is a fixed-purpose web-log
  analyser with HTML / terminal output. `rare` is generic;
  any text source with a regex extractable signal works
  (kernel logs, application traces, k8s events, app
  metrics emitted as text).

## Caveats

- **GPL-3.0.** Distributing a modified `rare` binary or a
  derivative work requires re-licensing under GPL-3.0;
  scripts that *call* `rare` and process its stdout are
  not derivatives.
- **Regex skill required.** Power comes from PCRE-shaped
  capture groups + the `{...}` expression language. There
  is a learning curve; `rare --help histo` and the docs at
  <https://rare.zdyn.net> are the on-ramp.
- **Terminal-only TUI.** Charts render with Unicode block
  characters; works in any modern terminal but not a great
  fit for static reports — pipe to `--json` for structured
  output instead.
- **Memory grows with cardinality.** `bars`, `tabulate`,
  and `heatmap` keep one bucket per distinct key; cap with
  `-n N` (top-N) when you suspect unbounded keys (raw IPs,
  request IDs, UUIDs).
- **No JOINs.** Each invocation reads its own input
  pipeline; there is no cross-stream join. For relational
  questions (this log + that log on user_id), reach for
  `mlr`, `q-text-as-data`, or `agrind`.
