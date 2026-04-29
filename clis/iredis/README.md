# iredis

> **An interactive Redis client with auto-completion,
> syntax highlighting, and live-rendered command help** —
> a `psql`-like REPL for Redis built on `prompt_toolkit`,
> the same library that powers [`pgcli`](../pgcli/) and
> [`mycli`](../mycli/). Pinned to **v1.16.0**
> ([LICENSE](https://github.com/laixintao/iredis/blob/master/LICENSE),
> BSD-3-Clause).

Source: <https://github.com/laixintao/iredis>

## TL;DR

`iredis` is what `redis-cli` would look like if it were
written today. The official `redis-cli` is a thin
TTY wrapper over the RESP protocol — no completion, no
syntax color, no inline docs, and any typo silently
returns `(error) ERR unknown command`. `iredis` keeps
the same connection model (`-h`, `-p`, `-n`, `-a`,
`--url redis://...`, TLS, sentinel, cluster) but layers
on a real REPL: as you type, it suggests the next token
from the full Redis command grammar (200+ commands across
strings, hashes, lists, sets, sorted sets, streams,
pubsub, scripting, modules); the bottom of the screen
shows the live syntax for the current command (`SET key
value [EX seconds] [PX milliseconds] [NX|XX]`); arguments
are color-coded by role (key vs. value vs. flag vs.
numeric); responses are pretty-printed (RESP3 maps as
indented JSON, byte strings as quoted-and-escaped strings,
binary data as hex on demand); and dangerous commands
(`KEYS *`, `FLUSHDB`, `FLUSHALL`, `DEBUG SLEEP`) prompt
for confirmation before being sent. Everything works
against any RESP2/RESP3 server — not just Redis OSS, but
Valkey, KeyDB, Dragonfly, and Upstash. Output is pipe-aware:
`echo "GET mykey" | iredis --no-raw` returns just the
value with no prompt furniture, so it slots cleanly into
shell pipelines and CI scripts.

## Install

```bash
# pipx (recommended — isolates from system Python)
pipx install iredis

# pip
pip install --user iredis

# Homebrew
brew install iredis

# Docker
docker run -it --rm --network host laixintao/iredis

# verify
iredis --version    # iredis, version 1.16.0
```

`iredis` is pure Python (3.8+), so no compile step is
needed. The binary is named `iredis`; on macOS / Linux
both `iredis` and `iredis-cli` are commonly aliased.

## Basic usage

```bash
# connect to localhost:6379, db 0
iredis

# connect via URL (TLS-aware)
iredis --url rediss://default:secret@cache.example.com:6380/2

# classic flags (drop-in for redis-cli)
iredis -h cache.example.com -p 6380 -n 2 -a "$REDIS_PASSWORD"

# read-only mode — disables every write command at the parser layer
iredis --readonly

# pipe a single command, get raw output, exit
echo "GET sessions:abc123" | iredis --no-raw

# script multiple commands
iredis < script.redis

# disable the dangerous-command prompt (CI use)
iredis --no-warning

# inside the REPL:
#   TAB              cycle completion candidates
#   F2               toggle live command-doc panel
#   F3               toggle key-completion (scans current db)
#   :help SET        inline doc for any command
#   :peek mykey      auto-detects type and dumps with appropriate command
#   :clear           clear screen, keep history
#   exit / Ctrl-D    quit
```

Configuration lives in `~/.iredisrc` (INI format) — set
default URL, color theme (Solarized / Monokai / etc.),
history file location, and which commands trigger the
confirmation prompt.

## When to choose

- **You debug Redis interactively more than once a week**
  — the gap between "type `KEYS sess:*` and instantly
  see colored, paginated, type-aware results" and
  "stare at `redis-cli` output that wraps at 80 columns
  and doesn't tell you which keys are hashes vs. lists"
  is large enough to justify the install on day one.
- **You operate against multiple Redis-compatible
  servers** (Redis OSS, Valkey, KeyDB, Dragonfly,
  Upstash) — `iredis` speaks the wire protocol cleanly
  and doesn't depend on Redis-OSS-specific features for
  its core REPL UX.
- **You want a safety net against finger-checks in
  production** — the built-in confirmation on
  `FLUSHALL` / `FLUSHDB` / `KEYS *` / `DEBUG SLEEP`
  has saved more than one on-call from a very bad
  evening. `--readonly` enforces this at the parser layer
  for read-only sessions on a primary.
- **You're already using [`pgcli`](../pgcli/) or
  [`mycli`](../mycli/) or [`litecli`](../litecli/)** —
  same `prompt_toolkit` foundation, same keybindings,
  same config file philosophy. Adding `iredis` to the
  set gives the same REPL ergonomics across all four
  databases.

## When NOT to choose

- **You only ever script Redis from application code** —
  if your interaction with Redis is purely through
  `redis-py` / `ioredis` / `lettuce`, an interactive
  REPL is dead weight. `redis-cli` is fine for the
  occasional `PING`.
- **You need a GUI for browsing keys and values** —
  reach for RedisInsight, Another Redis Desktop Manager,
  or `redli`'s web mode. `iredis` is a TUI / REPL only.
- **You're embedded in extreme-low-latency benchmarking**
  — the `prompt_toolkit` overhead adds a few ms per
  command vs. raw `redis-cli`. For benchmark loops use
  `redis-benchmark` or `memtier_benchmark`.
- **You need full Redis Streams consumer-group tooling**
  — `iredis` exposes the `XREADGROUP` / `XACK` /
  `XPENDING` commands but doesn't render consumer-group
  state graphically. Use `RedisInsight` or
  `redis-stream-cli` for that.

## Why it fits the zoo

The zoo's `dbcli` cluster — [`pgcli`](../pgcli/),
[`mycli`](../mycli/), [`litecli`](../litecli/),
[`harlequin`](../harlequin/), [`usql`](../usql/),
[`lazysql`](../lazysql/) — collects "what `psql` /
`mysql` / `sqlite3` would look like if rebuilt today
with completion, color, and live docs." `iredis` is
the same recipe applied to Redis, by an author
([@laixintao](https://github.com/laixintao)) who is
also a long-time `dbcli` family contributor — so the
keybindings, config conventions, and history-file layout
match the rest of that cluster. Adding it closes a
visible gap (the zoo had no Redis client) and gives
readers a coherent answer for "what's the
`prompt_toolkit`-style REPL for cache / KV stores?"

## Upstream pointers

- Repo: <https://github.com/laixintao/iredis>
- Release notes: <https://github.com/laixintao/iredis/releases>
- License: [BSD-3-Clause](https://github.com/laixintao/iredis/blob/master/LICENSE)
- Docs: <https://iredis.xbin.io/>
- Author: [@laixintao](https://github.com/laixintao)
  (also maintains [`china-ex`](https://github.com/laixintao/china-ex)
  and contributes to the `dbcli` org).
- Sibling tools: [`pgcli`](../pgcli/), [`mycli`](../mycli/),
  [`litecli`](../litecli/).
