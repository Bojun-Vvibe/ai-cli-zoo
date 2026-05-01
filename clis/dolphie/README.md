# dolphie

> **Real-time MySQL / MariaDB / ProxySQL TUI dashboard you can
> point at any host and read like a flight deck** — Textual-based
> Python app that connects over the standard MySQL protocol and
> renders panels for processlist, replication topology, InnoDB
> metrics, performance schema waits, locks, query digest, and
> per-thread history with delta deltas updated every second.
> Pinned to **v6.5.5** (released 2025-04-15,
> [LICENSE](https://github.com/charles-001/dolphie/blob/main/LICENSE),
> GPL-3.0).
>
> Source: <https://github.com/charles-001/dolphie>

## TL;DR

`dolphie` is what you reach for the moment a MySQL host starts
acting weird and the answer is not in Grafana yet. It is a
single-binary-feel Textual TUI that reuses the same
`information_schema` / `performance_schema` / `SHOW ENGINE INNODB
STATUS` queries you would otherwise paste into a `mysql` shell,
but presents them as live panels: top-N queries by latency,
running threads with full statement text, replication lag and
SBM, InnoDB row-lock waits and deadlock counter, buffer pool hit
rate, history of QPS / TPS / network / temp-tables. Every panel
has hotkeys to filter, sort, kill, and EXPLAIN — so the workflow
"see a slow thread → expand it → EXPLAIN → kill" stays inside one
window. Talks to MySQL 5.7+, MariaDB 10.4+, ProxySQL 2.x.

## Install

```bash
# pipx (recommended — isolated venv per app)
pipx install dolphie

# pip
pip install dolphie

# Homebrew (macOS / Linux)
brew install dolphie

# verify
dolphie --version    # Dolphie 6.5.5
```

## Examples

```bash
# connect to a host the same way mysql(1) would
dolphie -h db-prod-01.internal -u readonly -p

# read connection from a my.cnf section so creds stay out of argv
dolphie --login-path=client            # uses ~/.mylogin.cnf
dolphie --defaults-file=~/.my.cnf      # uses [client] section

# point at a ProxySQL admin port to inspect query rules + connection pool
dolphie -h proxysql-01 -P 6032 -u admin -p

# read-only safety: refuse to issue KILL even if you fat-finger the hotkey
dolphie -h db-prod-01 -u readonly -p --read-only

# replay a saved snapshot (great for post-incident review or a screenshot)
dolphie --record-file incident.log -h db-prod-01 -u readonly -p
dolphie --replay-file incident.log

# inside the TUI, common hotkeys:
#   p   processlist        r  replication        i  innodb status
#   l   locks              q  query digest       D  dashboard
#   k   kill thread        e  EXPLAIN selected   /  filter
```

## Use when

- A production MySQL is **acting up right now** — connections
  spiking, replica lag climbing, locks accumulating — and you
  want one screen that updates every second instead of running
  `SHOW PROCESSLIST` in a loop and squinting at the diff.
- You need a **session-scoped flight recorder** of an incident:
  `--record-file` captures the metric stream, `--replay-file`
  lets you scrub back through it during the post-mortem without
  the live host changing under you.
- You operate **ProxySQL** in front of MySQL and want one tool
  that flips between the backend (real DB threads, InnoDB stats)
  and the proxy (rule hits, hostgroups, connection pool) using
  the same TUI grammar.
- You want a **read-only safe mode** to hand to an on-call
  engineer who needs visibility but should not be one keystroke
  away from `KILL`-ing the wrong session — `--read-only` disables
  every mutating action.
- Pair with [`mycli`](../mycli/) (interactive
  REPL with auto-completion for the actual SQL you decide to
  run after dolphie shows you what is hot),
  [`pgcli`](../pgcli/) (Postgres sibling of mycli — different
  engine, same ergonomic),
  [`harlequin`](../harlequin/) /
  [`lazysql`](../lazysql/) (cross-engine TUI clients for
  ad-hoc queries against the same host),
  [`gh-dash`](../gh-dash/) /
  [`k9s`](../k9s/) (other Textual-style live TUIs you can leave
  open in adjacent panes during an incident).

Skip `dolphie` for **Postgres** (use
[`pgcli`](../pgcli/) +
[`pg_top`](https://pg_top.gitlab.io/) /
your APM); dolphie is MySQL / MariaDB / ProxySQL only. Skip if
your operational story is "everything must go through Datadog /
Grafana / PMM" — dolphie is point-in-time tactical visibility,
not long-term metric storage. Not a replacement for a query
analyzer like Percona PMM's QAN; it shows what is happening *now*
on the box.
