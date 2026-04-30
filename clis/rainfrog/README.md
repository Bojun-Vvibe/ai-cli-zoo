# rainfrog

> **A keyboard-driven database management TUI** —
> connect to PostgreSQL, MySQL, SQLite, or
> MS SQL from one Rust binary, browse schemas in
> a tree pane, run queries in a vim-style editor,
> and page through results without leaving the
> terminal. Pinned to **v0.3.18**
> ([LICENSE](https://github.com/achristmascarl/rainfrog/blob/main/LICENSE),
> MIT).

Source: <https://github.com/achristmascarl/rainfrog>

## TL;DR

`rainfrog` is the "I live in tmux and need to
poke a database" tool. Launch it with a
connection string (or pick a saved profile from
`~/.config/rainfrog/config.json`), and you get a
three-pane Ratatui interface: a schema tree on
the left, a query editor in the top right, and
a paginated result grid on the bottom right.
The editor honors **vim modal keybindings** out
of the box (`i` to insert, `:w` to save the
buffer to a `.sql` file, `Ctrl-Space` to run),
and queries that return wide tables get a
column-pinning shortcut so you can keep `id` and
`created_at` visible while scrolling. It speaks
**Postgres, MySQL/MariaDB, SQLite, and SQL
Server** through `sqlx`, so the same binary
covers most working setups; connection
favorites land in a small JSON file you can
commit (without secrets — passwords resolve
from `$DATABASE_URL` or a keyring entry at run
time). Schema introspection includes indexes,
foreign keys, and views, and `Ctrl-D` on a row
opens a "describe row" panel that resolves FK
references to neighbour tables one keypress
away.

## Install

```bash
# Homebrew
brew install rainfrog

# Cargo
cargo install rainfrog --locked

# prebuilt binaries (Linux / macOS / Windows)
# https://github.com/achristmascarl/rainfrog/releases

# verify
rainfrog --version    # rainfrog 0.3.18
```

## Basic usage

```bash
# Postgres via DATABASE_URL
export DATABASE_URL='postgres://app:secret@localhost:5432/app_dev'
rainfrog

# explicit driver + connection string
rainfrog --driver postgres --url 'postgres://...'

# SQLite file
rainfrog --driver sqlite --url 'sqlite://./app.db'

# launch on a saved connection profile
rainfrog --connection staging-replica
```

## When to choose

- **You need a real query editor without a
  desktop GUI** — `psql` and `sqlite3` are great
  for one-liners but painful for iterating on a
  20-line CTE; rainfrog gives you vim-mode
  editing, multi-statement buffers, and result
  paging in the same pane.
- **You hop between Postgres and SQLite (or
  add MySQL / SQL Server) and want one tool**
  — the keybindings, schema browser, and result
  grid are identical across drivers, so muscle
  memory carries over.
- **You SSH into a jump host and run queries
  there** — single static Rust binary, no JVM,
  no Electron, works fine over a 500ms link.

## When NOT to choose

- **You need migrations / DDL diffing** — use
  [`atlas`](../atlas/), [`sqitch`](../sqitch/),
  or [`dolt`](../dolt/); rainfrog is a query
  client, not a schema-management tool.
- **You want CSV-shaped exploration of any
  tabular file** — use [`visidata`](../visidata/)
  or [`tabiew`](../tabiew/); rainfrog is
  database-connected, not file-driven.
- **You want a non-interactive query runner
  for scripts** — use [`pgcli`](../pgcli/) /
  [`iredis`](../iredis/) / native CLI clients;
  rainfrog is a TUI, not a pipe-friendly
  command.

## Why it fits the zoo

Slots into the "small Rust TUI for a
specific datastore" cluster
([`lazysql`](../lazysql/),
[`pgcli`](../pgcli/), [`iredis`](../iredis/),
[`atac`](../atac/)). rainfrog's specific gap is
**multi-driver coverage with vim-mode SQL
editing** — lazysql is close on multi-driver
but uses a custom edit mode; pgcli is
single-driver REPL; iredis is Redis-only.

## Upstream pointers

- Repo: <https://github.com/achristmascarl/rainfrog>
- Release notes: <https://github.com/achristmascarl/rainfrog/releases>
- License: [MIT](https://github.com/achristmascarl/rainfrog/blob/main/LICENSE) (`LICENSE`)
- Maintainer: [@achristmascarl](https://github.com/achristmascarl)
