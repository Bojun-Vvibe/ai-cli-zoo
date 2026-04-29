# lazysql

- **Repo:** https://github.com/jorgerojas26/lazysql
- **Version:** v0.4.8 (commit `075b18df9f3556c1a4603d0ea8da9410699924c5`,
  released 2026-02-17)
- **License:** MIT ([LICENSE.txt](https://github.com/jorgerojas26/lazysql/blob/main/LICENSE.txt))
- **Language:** Go (built on `tview` / `tcell`)
- **Install:** `brew install lazysql` · `go install github.com/jorgerojas26/lazysql@latest` ·
  pre-built binaries on the GitHub releases page · Arch AUR (`lazysql`)

## What it does

`lazysql` is a cross-platform terminal database client that takes the
[`lazygit`](../lazygit/) / [`lazydocker`](../lazydocker/) layout — a
left-hand object tree, a tabbed work area on the right, single-letter
keybindings — and applies it to relational databases. One binary speaks
**MySQL / MariaDB, PostgreSQL, SQLite, and SQL Server** through native Go
drivers (no ODBC layer, no JDBC bridge, no Python runtime), with connection
profiles persisted to `~/.config/lazysql/config.toml` so opening a saved
connection is a `j`/`k` + `Enter` away. Inside an open connection you get
a sidebar of databases → schemas → tables, a results pane that pages
through rows with arrow keys (no `LIMIT 1000` mental overhead), an
**inline cell editor** (move to a cell, press `c`, type a new value,
`Ctrl+S` queues the `UPDATE`, `Ctrl+R` runs all queued mutations as a
single transaction) so quick data fixes don't need a hand-written `UPDATE
… WHERE id=…`, a `Ctrl+Space` SQL editor tab with multi-statement
execution and result tabs per query, and a `Ctrl+E` table-DDL viewer
that prints the `CREATE TABLE` reconstructed from `information_schema`.
Foreign-key columns are clickable — `Enter` on the FK value navigates to
the referenced row in the parent table, and a back-stack (`Esc`) walks
you back, which makes ad-hoc data exploration of a normalised schema feel
like clicking through links rather than writing joins.

## When to pick it / when not to

Pick `lazysql` when you live in the terminal and want an *interactive*
data-exploration UI — the cell editor + FK-navigation combo is what
separates it from CLI shells like [`pgcli`](../pgcli/) /
[`mycli`](../mycli/) / [`litecli`](../litecli/) (those are excellent
REPLs but you still hand-write every navigation query). Pick it when
your week routinely touches more than one of {Postgres, MySQL,
SQLite, SQL Server} and you'd rather have one keymap than four. Pairs
naturally with [`harlequin`](../harlequin/) (analytical TUI for
DuckDB / Snowflake / BigQuery / Trino — different engines, similar
spirit), [`usql`](../usql/) (universal SQL CLI for non-interactive
scripting / migrations), and [`atlas`](../atlas/) /
[`dbmate`](../dbmate/) for schema migration alongside ad-hoc edits.

Skip it for engines outside its driver set (DuckDB, Snowflake, BigQuery,
Oracle, ClickHouse, MongoDB — use [`harlequin`](../harlequin/),
[`usql`](../usql/), or the engine-native client). Skip it for
non-interactive scripting and CI — it's a TUI, not a batch tool; reach
for `psql -c` / `mysql -e` / `usql -c` / `sqlite3 -cmd`. Skip it if your
team's database-of-record is production and your fingers slip — the
inline editor is fast precisely because it does not double-confirm every
mutation; point it at staging / a replica / a local snapshot and use
[`atlas`](../atlas/) /
[`dbmate`](../dbmate/) /
[`golang-migrate`](../golang-migrate/) for production change control.
Project is pre-1.0; pin the version and read release notes before bumping.

## Example invocations

```bash
# First run — opens a connection-manager screen
lazysql

# Or pass a DSN directly to skip the connection picker
lazysql "postgres://user:pass@localhost:5432/mydb?sslmode=disable"
lazysql "mysql://user:pass@tcp(localhost:3306)/mydb"
lazysql "sqlite3:///path/to/file.db"
lazysql "sqlserver://user:pass@localhost:1433?database=mydb"

# Common keystrokes inside the TUI:
#   ↑/↓ or j/k    navigate sidebar / rows
#   Enter         open table / follow FK link
#   Esc           back-stack (return to previous table view)
#   c             edit cell under cursor
#   d             delete row
#   o             insert (add) row
#   Ctrl+S        stage edited cell as a pending mutation
#   Ctrl+R        run all pending mutations in one transaction
#   Ctrl+Z        discard pending mutations
#   Ctrl+E        show CREATE TABLE for the current table
#   Ctrl+Space    open the multi-statement SQL editor in a new tab
#   /             filter / search rows in the current result set
#   ?             show full keybinding help
#   q             quit

# Verify
lazysql --version    # lazysql v0.4.8
```
