# pgcli

> **Postgres REPL with live schema-aware autocomplete and syntax
> highlighting** — a `prompt_toolkit`-based interactive client for
> PostgreSQL / Greenplum / Amazon Redshift that completes table
> names, column names, function names, keywords, aliases, and
> `JOIN` conditions from the *currently connected* schema (refreshed
> in the background so newly-created tables appear without
> reconnecting), syntax-highlights the buffer as you type,
> auto-formats on `Tab`, supports proper multi-line editing with
> indent, pages results through `less`, and stores per-host history
> + named-query favourites in `~/.config/pgcli/config`. Pinned to
> **v4.4.0** (commit `e92db29e2c9c54226e3e8341357fcb514de70ea1`,
> [LICENSE.txt](https://github.com/dbcli/pgcli/blob/main/LICENSE.txt),
> BSD-3-Clause).

Source: <https://github.com/dbcli/pgcli>

## TL;DR

`pgcli` is what you reach for when stock `psql` is fine but the
"type `\d users` in another tab to remember the column name"
detour is breaking your flow. The completer asks the live catalog,
so on a fresh connection to a 400-table schema it already knows
every table, every column, every index, every function — `SELECT `
followed by `Tab` lists the columns of whatever table you mention
in the `FROM` clause. `\T markdown` switches output to a paste-
ready table; `\timing` toggles per-query wall time; `\fs name <sql>`
saves a favourite that `\f name` re-runs. Same author / same UX
as the [`mycli`](../mycli/) (MySQL) and `litecli` (SQLite)
siblings, so the muscle memory is portable across all three engines.

## Install

```bash
# Homebrew (macOS / Linux)
brew install pgcli

# pipx (recommended for Python users — isolated venv)
pipx install pgcli

# pip
pip install --user pgcli

# Debian / Ubuntu
sudo apt install pgcli

# Fedora
sudo dnf install pgcli
```

## Usage

```bash
# Connect using a libpq URI (env vars / .pgpass also honoured)
pgcli postgresql://app_ro:$PG_PASS@db.internal:5432/orders

# Once at the prompt:
#   \dt              list tables
#   \d users         describe one table
#   \T markdown      switch result rendering to markdown
#   \timing          toggle per-query timing
#   \fs top10 SELECT id, total FROM orders ORDER BY total DESC LIMIT 10;
#   \f top10         re-run the saved favourite
#   \x               toggle expanded (one-column-per-line) display
#   \?               full backslash-command help
#   Ctrl+R           fzf-style reverse history search across past sessions
```

## Why pick this

Stock `psql` is rock-solid but completion is keyword-only and there
is no syntax highlighting; `pgcli` keeps `psql`'s `\` command set
(so existing scripts and habits transfer) and layers on the
Postgres-aware completion + highlighting + named queries that
remove most of the "open another tab to remind myself of the
schema" friction. For an LLM-CLI workflow, `pgcli --help-with -e
"<sql>"` runs one statement non-interactively and returns the
result on stdout in a chosen format (CSV / TSV / markdown / JSON
via `\T`), which is the right shape for piping into
[`jq`](../jq/) / [`miller`](../miller/) downstream.
