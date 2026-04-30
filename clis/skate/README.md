# skate

- **Repo:** https://github.com/charmbracelet/skate
- **Latest version:** v1.0.1 (released 2025-03-06)
- **License:** MIT — SPDX `MIT` — [LICENSE](https://github.com/charmbracelet/skate/blob/main/LICENSE)
- **Category:** personal key-value store CLI (local Badger DB, optional sync)

A **single Go binary that gives you a personal, scriptable key-value
store on the command line.** `skate set greeting hello` writes,
`skate greeting` reads, `skate list` enumerates, `skate delete
greeting` removes, `skate ls -k` lists keys only. Values can be
arbitrary bytes (read from STDIN if omitted: `cat config.json | skate
set @work cfg`), keys can be namespaced into separate "databases"
(`skate set foo bar@notes`), and the storage backend is a local
embedded `dgraph-io/badger` LSM-tree under
`~/Library/Application Support/skate/` (macOS) or
`$XDG_DATA_HOME/skate/` (Linux), so reads and writes are sub-millisecond
and there is no daemon. Originally shipped with optional end-to-end
encrypted sync against the (now-sunset) Charm Cloud; v1.0.1 keeps the
local store as the supported surface and the README documents the
Charm Cloud sunset.

## Why it's interesting

Shell scripts and AI-CLI workflows constantly need **a small piece of
state that should persist across invocations and is too transient for
a config file**: the last branch you reviewed, a counter for an
overnight loop, a per-project API base URL, a cached nonce. The
options are usually (a) a flat file you reinvent the locking on, (b)
sqlite via `sqlite3` shell-out, or (c) a real KV server. `skate`
collapses the gap: one binary, no schema, no daemon, namespaces per
project (`@projectname`), STDIN-aware, exit-code-clean, scripts like
`API_KEY=$(skate api_key@prod)` work everywhere. Pairs naturally with
the rest of the Charm CLI suite ([`gum`](../gum/), [`glow`](../glow/),
[`vhs`](../vhs/)) since the API surface and binary style match.

## Install

```bash
# macOS / Linux
brew install charmbracelet/tap/skate

# Go toolchain (any platform)
go install github.com/charmbracelet/skate@latest

# Prebuilt binary
curl -L https://github.com/charmbracelet/skate/releases/download/v1.0.1/skate_1.0.1_Linux_x86_64.tar.gz | tar xz
sudo mv skate /usr/local/bin/
```

## Sample invocation

```bash
# Write + read a simple value
skate set api_base https://api.example.com
skate api_base
# => https://api.example.com

# Per-project namespace, value from STDIN
gh auth token | skate set token@work
TOKEN=$(skate token@work)

# Enumerate keys in a namespace
skate list @work

# Delete
skate delete token@work
```

## Comparable to

- `redis-cli` against a local `redis-server` — heavier, needs a
  daemon, but full Redis semantics.
- `sqlite3 ~/state.db "..."` — more powerful (real SQL) but every
  script reinvents the schema and the locking.
