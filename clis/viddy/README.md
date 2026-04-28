# viddy

- **Repo:** https://github.com/sachaos/viddy
- **Version:** v1.3.0 (latest stable, 2024-11)
- **License:** MIT ([LICENSE](https://github.com/sachaos/viddy/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install viddy` · `go install github.com/sachaos/viddy@latest` · binary releases on the GitHub release page

## What it does

`viddy` is a modern, "pimp-my-`watch`" replacement: it re-runs a command on
a fixed interval and renders the output, but with three things classic
`watch(1)` doesn't have. **(1) Time-machine mode** — every execution is kept
in an in-memory ring buffer, so you can press `Space` to pause, then
`Shift+J/K` (or arrow keys) to scrub backwards through previous executions
and watch how the output evolved. **(2) Diff highlighting** — `-d` (or `D`
toggled live) marks added / changed bytes between consecutive runs in
colour, so a slowly-growing counter or a flapping pod jumps out instantly.
**(3) Real TUI controls** — pageable / scrollable output (so `viddy -n1
'kubectl get pods -A'` doesn't truncate at terminal height the way `watch`
does), search within the buffer (`/`), help overlay (`?`), and a "show all
historical runs as a stacked bar of deltas" view (`s`). Configurable via
`~/.config/viddy.toml` (interval, shell, colours, keybinds, ring buffer
size).

## When to pick it / when not to

Pick `viddy` when you would otherwise reach for `watch -n1 -d 'something'`
and immediately wish you could (a) scroll back five seconds to see the
output that flashed by, (b) page through output longer than your terminal,
or (c) see *which* bytes changed across runs, not just whether anything
changed. Canonical use cases: tailing a slowly-converging deploy
(`kubectl rollout status`, `kubectl get pods`), watching a metric endpoint
(`curl -s localhost:9090/metrics | grep my_counter`), monitoring a build
queue, watching `df -h` during a long copy, or polling a CI status endpoint.
The diff view turns "did anything just change?" from a staring contest into
a glance.

Skip it for one-shot commands (just run them), for commands that produce
binary or escape-heavy output (the diff view will be noisy), or for
sub-second polling of expensive commands — the default 2 s interval exists
for a reason. For long-term metric watching, a real time-series tool
(Prometheus + Grafana, `bottom`, [`btop`](../btop/)) beats polling. And
remember the buffer is in-memory: closing `viddy` discards history.

## Example invocations

```bash
# Re-run every 1 s, highlight diffs vs previous run
viddy -n 1 -d 'kubectl get pods -A'

# Watch a Prometheus counter, diff mode on, time-machine accessible
viddy -n 2 -d "curl -s localhost:9090/metrics | grep ^http_requests_total"

# Pause anytime with <Space>, scrub history with <Shift+J>/<Shift+K>,
# search within a frame with /, toggle diff with d, quit with q

# Poll a build until it stops being "running"
viddy -n 5 'gh run list --limit 5'

# Use a non-default shell so pipelines / globs work as expected
viddy --shell bash -n 1 'ls -la *.log | head -20'
```
