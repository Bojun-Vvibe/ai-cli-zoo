# harlequin

> The **SQL IDE for your terminal**: a TUI that opens DuckDB / Postgres /
> SQLite / MySQL / Snowflake / BigQuery, gives you autocomplete + a
> schema tree + a results grid, and stays out of your way. Pinned to
> **v2.5.2** (commit `cbaccc98c669c7f7a1b2b35268a00c726f20431b`,
> [LICENSE](https://github.com/tconbeer/harlequin/blob/main/LICENSE), MIT).

Source: <https://github.com/tconbeer/harlequin>

## TL;DR

`harlequin` is `psql` / `duckdb` shell, but with the ergonomics of
DataGrip in 80×24. `harlequin my.duckdb` opens a Textual TUI with three
panes: a schema tree on the left (databases → schemas → tables → columns,
all lazy-loaded), a SQL editor in the middle (multi-buffer tabs,
autocomplete from the live catalog, syntax highlighting, `Ctrl+Enter` to
run), and a paginated results grid on the bottom (arrow-key navigable,
`Ctrl+E` to export selection to CSV / Parquet / JSON). Adapters for
Postgres, SQLite, MySQL, Snowflake, BigQuery, Trino, MotherDuck, and
ODBC are separate `pip install` extras. The whole thing is one Python
process, one config file (`~/.config/harlequin/config.toml`), and SSHs
cleanly to any server with Python 3.9+.

## Install

```bash
# DuckDB (default adapter, bundled)
pip install harlequin
# or with uv
uv tool install harlequin

# add adapters as needed
pip install 'harlequin[postgres]'
pip install 'harlequin[mysql]'
pip install 'harlequin[snowflake]'
pip install 'harlequin[bigquery]'

# verify
harlequin --version
```

Python 3.9+. Pure Python, no native deps beyond what each adapter pulls.

## One Concrete Example

```bash
# DuckDB on a local Parquet directory
harlequin -t monokai 'data/*.parquet'

# Postgres
harlequin -P postgres \
  --host db.internal --port 5432 --user analyst --database warehouse

# Run a script non-interactively (CI / cron)
harlequin --no-tui -e "select count(*) from events where ts > now() - interval '1 day'"

# Inside the TUI
#   Ctrl+Enter   run current statement
#   F2           rename buffer tab
#   Ctrl+E       export results (csv / parquet / json)
#   F1           help / keymap
#   Ctrl+Q       quit
```

A first session typically looks like: open the file, expand the schema
tree until you find the table, hit `Ctrl+Space` in the editor to
autocomplete a `SELECT * FROM <tab>`, run with `Ctrl+Enter`, then
`Ctrl+E` to dump the result grid to `out.parquet` for the next step.

## Niche It Fills

**A real SQL IDE inside `tmux` / SSH.** Everything else in the catalog
that touches data either (a) wraps SQL in an LLM (`vanna`, `wrenai`,
`db-gpt`) or (b) is a one-shot CLI (`sqlite-utils`, `datasette`).
`harlequin` is the missing piece in the middle: an interactive SQL
editor with multi-engine autocomplete that doesn't require X11, a
browser, or a JetBrains license. It pairs naturally with the LLM SQL
tools — let `vanna` draft the query, paste it into `harlequin` to
inspect the plan and the rows.

## Vs Already Cataloged

- **Vs [`datasette`](../datasette/):** `datasette` is a
  read-only HTTP / web UI for SQLite — great for sharing data with a
  browser audience. `harlequin` is a write-capable terminal IDE for
  the operator. Different users, different surfaces.
- **Vs [`vanna`](../vanna/) / [`wrenai`](../wrenai/):** those generate
  SQL from natural language; `harlequin` is where you *land* that SQL
  to actually run, edit, and inspect it. Use them together.
- **Vs raw `duckdb` / `psql`:** the bundled REPLs have no schema tree,
  no multi-buffer editor, no results pagination, and no cross-engine
  parity. `harlequin` is what those shells would be if they had been
  designed in 2024.

## Caveats

- Adapters are extras — `pip install harlequin` alone gets you DuckDB
  only. A "no Postgres driver" error on first connect almost always
  means you skipped `pip install 'harlequin[postgres]'`.
- The TUI assumes a terminal with at least 80 columns and true-color
  support. It runs in `xterm-256color`, but the schema tree gets
  cramped under ~100 cols.
- `--no-tui -e "..."` is the headless mode for scripts, but it still
  opens the adapter connection — for sub-second one-shots, prefer
  `duckdb -c "..."` or `psql -c "..."` directly.
- BigQuery / Snowflake adapters use those vendors' Python clients, so
  auth is whatever those clients expect (ADC, `~/.snowflake/config`,
  etc.); `harlequin` itself stores no credentials.
