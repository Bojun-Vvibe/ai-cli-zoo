# wtfutil

## What it does
A **personal terminal dashboard** that tiles a configurable grid of widgets
into one full-screen TUI: clocks across timezones, GitHub / GitLab / Gitea
PR queues, Jira / Trello / Asana / GitHub Issues backlogs, calendar (Google,
ICS), CI status (CircleCI, BambooHQ, Gitlab CI, Jenkins), Datadog / New
Relic / Grafana / Prometheus / Zabbix alerts, AWS / DigitalOcean / GCloud
inventory, weather, RSS, IPinfo, Pocket, Todoist, Hibp, Krisinformation,
RSS feeds, hacker news, exchange rates, OpsGenie, PagerDuty, plus
arbitrary `cmdrunner` widgets that just shell out and render the stdout.
Layout is a YAML grid (`~/.config/wtf/config.yml`) that addresses widgets by
`top`/`left`/`width`/`height` cells, each widget polls on its own
`refreshInterval`, and a single keypress focuses a widget for keyboard
navigation. One static Go binary, no daemon, no server side.

## Why it's interesting
Different shape from a browser tab full of dashboards (heavy, mouse-driven,
needs SSO every morning), from `tmux` + a fan of bespoke shell loops
(works but every team rewrites the same widgets), from `gh dash` /
[`gh-dash`](../gh-dash/) (excellent but GitHub-PR-shaped only), from
[`k9s`](../k9s/) / [`lazygit`](../lazygit/) / [`btop`](../btop/) (single-
domain TUIs), and from Grafana / Datadog dashboards (web, server-side,
ops-focused). wtfutil is the *personal morning-standup dashboard* shape:
pick it specifically when the ask is "one terminal pane that shows my PRs +
my Jira + my on-call + my calendar + the build status I care about, side
by side, refreshing on its own, over SSH, with no browser." Do **not**
pick it for production observability (use Grafana / Datadog), for team-
shared status pages (use a real status board), or when the only widget you
need is GitHub PRs (use [`gh-dash`](../gh-dash/) — lighter and PR-native).

## Niche category
Personal terminal dashboard — YAML-tiled multi-widget TUI aggregating PR /
issue / CI / calendar / monitoring sources in one screen.

## Repo
https://github.com/wtfutil/wtf

## Version pinned
`v0.49.1` (latest tagged release on the `trunk` branch, 2026-04-17)

## License
- SPDX: `MPL-2.0`
- License file in upstream repo: `LICENSE.md`

## Install
```sh
# Homebrew
brew install wtfutil

# Go (from source)
go install github.com/wtfutil/wtf@latest

# Pre-built binaries:
# https://github.com/wtfutil/wtf/releases/latest

# Arch
sudo pacman -S wtfutil
```

## Usage examples
```sh
# First run creates ~/.config/wtf/config.yml with a starter grid
wtfutil

# Point at a custom config file
wtfutil --config=/path/to/dashboard.yml

# Inside the TUI:
#   Tab        cycle focus through widgets
#   /          focus mode help
#   Esc        leave focus mode
#   r          force-refresh focused widget
#   q          quit

# Minimal config sketch (~/.config/wtf/config.yml):
#   wtf:
#     grid:
#       columns: [40, 40, 40]
#       rows:    [10, 10, 10]
#     mods:
#       clocks:
#         enabled: true
#         position: { top: 0, left: 0, height: 1, width: 1 }
#         sleepCycles: 60
#         locations:
#           Local:   "Local"
#           NewYork: "America/New_York"
#           Berlin:  "Europe/Berlin"
#       githubpr:
#         enabled: true
#         position: { top: 1, left: 0, height: 1, width: 2 }
#         repositories:
#           someorg/somerepo: ["main"]
```
