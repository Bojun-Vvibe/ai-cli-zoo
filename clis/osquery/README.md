# osquery

## What it does
Exposes the operating system as a high-performance relational database. Running processes, listening sockets, kernel modules, installed packages, scheduled tasks, browser plugins, file hashes, and hundreds of other host facts each become a SQL table queryable from `osqueryi` (interactive shell) or `osqueryd` (long-running daemon that runs scheduled query packs and ships diff-results to a logger). Same query language works on Linux, macOS, Windows, and FreeBSD.

## Why it's interesting
The defensive-security and fleet-observability primitive: instead of writing per-OS shell scripts to enumerate host state, you write one `SELECT` and ship it to every endpoint. Query packs from the `osquery` and `palantir/osquery-configuration` repos cover CIS benchmarks, persistence-mechanism hunting, and incident-response triage. Pairs naturally with a log pipeline (Fluent Bit, Vector) for centralized fleet introspection — read-only, no payload execution, no remote shell.

## Niche category
Endpoint observability — SQL-on-OS / fleet introspection

## Repo
https://github.com/osquery/osquery

## Version pinned
`5.22.1`

## License
- SPDX: `Apache-2.0` (with GPL-2.0 alternative; user picks one at distribution time)
- License file in upstream repo: `LICENSE` (notice), `LICENSE-Apache-2.0`, `LICENSE-GPL-2.0`

## Install
```sh
# macOS
brew install --cask osquery

# Linux (Debian/Ubuntu) — official package repo
curl -L https://pkg.osquery.io/deb/pubkey.gpg | sudo apt-key add -
sudo add-apt-repository 'deb [arch=amd64] https://pkg.osquery.io/deb deb main'
sudo apt-get update && sudo apt-get install osquery
```

## Usage examples
```sh
# Interactive SQL shell against the live OS
osqueryi "SELECT pid, name, path FROM processes WHERE on_disk = 0;"

# Listening sockets joined to the owning process
osqueryi "SELECT p.pid, p.name, l.address, l.port
          FROM listening_ports l JOIN processes p ON l.pid = p.pid;"

# Run as a daemon with a scheduled query pack
sudo osqueryd --config_path=/etc/osquery/osquery.conf
```
