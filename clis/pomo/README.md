# pomo

> Snapshot date: 2026-05. Upstream: <https://github.com/kevinschoon/pomo>

**A Pomodoro timer that lives in your terminal and your status bar.**
`pomo` is a tiny Go CLI that runs a single SQLite-backed daemon, manages
named Pomodoro tasks (work intervals + breaks), and exposes the current
state on a Unix socket so any status-bar (tmux, i3bar, polybar, sketchybar,
xmobar) can render the remaining time without polling. It's the
"keep-a-25-min-timer-and-actually-write-down-what-I-did" tool, with
history persisted in `~/.pomo/pomo.db` so you can answer
"what did I work on last Tuesday?" three weeks later.

## Repo + version + license

- Repo: <https://github.com/kevinschoon/pomo>
- Latest release: **`0.8.1`** (2022-05-30)
- HEAD on `master`: `56f1659`
- License: **MIT** —
  <https://github.com/kevinschoon/pomo/blob/master/LICENSE>
- License path in repo: `LICENSE` (SPDX: `MIT`)
- Default branch: `master`
- Language: Go (SQLite via `mattn/go-sqlite3`, terminal UI via `gizak/termui`)

## Install

```bash
# Go install (most reliable on macOS / Linux)
go install github.com/kevinschoon/pomo@latest

# Arch (AUR)
yay -S pomo

# From source
git clone https://github.com/kevinschoon/pomo && cd pomo && make build
```

On first run, `pomo init` creates `~/.pomo/pomo.db` and a default config.

## Hello-world usage

```bash
# One-shot: a single 25-minute Pomodoro labelled "spec review"
pomo start -d 25m "spec review"

# Classic 4 x (25m work + 5m break) cycle for a longer task
pomo start -d 25m -p 4 "refactor cli-zoo dispatcher"

# What's running right now? (pretty terminal UI: bar, ticker, task name)
pomo status

# What's running right now? (one-line JSON, perfect for tmux/i3bar)
pomo status --format json

# Pause / resume the current Pomodoro
pomo pause   # Ctrl-Z for your brain
pomo start   # resume

# History: list every completed Pomodoro
pomo list

# Tag-aware history (tags are added at start time with -t)
pomo start -d 25m -t deep-work -t ai-cli-zoo "draft 3 entries"
pomo list -t deep-work --since 7d
```

Drop the JSON status into a tmux status line:

```tmux
set -g status-right "#(pomo status --format json | jq -r '.message // \"idle\"') | %H:%M"
```

## Niche

The "**Pomodoro with a queryable history, not a popup**" slot.

- GUI Pomodoro apps (Be Focused, Forest, Pomotroid) — pretty, but
  history lives in a proprietary SQLite blob you can't `jq` and they
  steal focus when the timer fires.
- [`porsmo`](../porsmo/) — modern Rust TUI Pomodoro; great for the
  in-flight experience, but no daemon and no persistent history layer.
- [`countdown`](../countdown/), [`termdown`](../termdown/) — generic
  countdown TUIs; no concept of a "task" or a session log.
- [`watson`](https://github.com/jazzband/Watson),
  [`taskwarrior`](../taskwarrior/) + `timew` ([`timew`](../timew/)) —
  excellent time trackers, but they bill *intervals*, not Pomodoros,
  and don't enforce the 25/5 rhythm.

`pomo` sits between "just a timer" and "full time tracker": it enforces
the Pomodoro discipline (work blocks of fixed length, mandatory breaks),
but everything is durable in SQLite so you can later answer "how many
Pomodoros did I actually finish on this project?" with a SQL query.

## Why it matters

- **Daemonised + status-bar-ready.** Most CLI Pomodoros are a
  foreground `sleep 1500 && say done`. `pomo` runs a real daemon,
  exposes structured state on a socket, and renders cleanly in
  any status bar via `pomo status --format json`. You can close the
  terminal you started it in.
- **Tags + SQLite = honest weekly review.** Every Pomodoro is a row;
  tags are first-class. Want "hours of deep-work tagged `ai-cli-zoo`
  this month"? It's a `SELECT` away (the schema is documented in the
  repo).
- **No accounts, no cloud, no telemetry.** All data is in
  `~/.pomo/pomo.db`. Backed up by anything that backs up `~`.
- **Unix-shaped.** Sub-commands compose with `jq`, `xargs`, `cron`,
  and your shell prompt. `pomo status --format json` is a stable
  interface — you can pipe it into mood-tracking, daily standup
  generators, or a "did I forget to take a break?" cron job.
- **Tiny and stable.** Last release is `0.8.1` from 2022 — not because
  it's abandoned but because the surface area is *done*: a Pomodoro is
  a Pomodoro. The Go binary is ~8 MB stripped and the SQLite schema
  hasn't needed to change.
