# ticker

- **Repo:** https://github.com/achannarasappa/ticker
- **Latest version:** v5.2.1
- **License:** GPL-3.0 (`LICENSE`)
- **Category:** finance / terminal-dashboard

A terminal-based stock ticker and watchlist dashboard, written in Go.
Configure your holdings (or just symbols) in a YAML file and `ticker`
streams real-time quotes from Yahoo Finance, computing day change,
total gain/loss, position value, weighted cost basis, and portfolio
sums — all in a refreshing TUI that fits on a tmux pane. Supports
multiple watchlists, currency conversion, after-hours quotes, and
custom column layouts (sort by change, alphabetical, value, etc.).
No account, no API key, no broker integration: it's a read-only
quote tape, deliberately scoped to "what's my screen showing right
now" rather than trade execution.

## Install

```bash
# Homebrew
brew install ticker

# Go
go install github.com/achannarasappa/ticker@latest

# Docker
docker run --rm -it -v $(pwd)/.ticker.yaml:/.ticker.yaml achannarasappa/ticker

# verify
ticker --version
```

## Why it's interesting

Most "portfolio tracker" tools want an account, a phone app, and your
brokerage credentials. `ticker` is the opposite: a single config file,
a terminal pane, no telemetry, no login. Pairs naturally with `tmux`
or `zellij` as a glanceable side-panel during the trading day, and
the YAML-as-config model means watchlists are version-controllable.
