# gobang

- **Repo:** https://github.com/TaKO8Ki/gobang
- **Latest version:** 0.1.0-alpha.5
- **License:** MIT (`LICENSE`)
- **Category:** dev-tools / database-tui

A cross-platform terminal UI for browsing MySQL, PostgreSQL, and SQLite
databases, written in Rust on top of `tui-rs` and `sqlx`. Connect with a
single command, then navigate databases / tables / records with vim-style
keybindings, inspect schema, run ad-hoc SQL, and filter result sets without
leaving the terminal. Pick gobang over `mycli` / `pgcli` when you want a
panel-based explorer rather than a REPL — useful for poking at an unfamiliar
database, eyeballing foreign-key relationships, or doing read-mostly
debugging over SSH. Connection strings live in a YAML config so switching
between dev / staging / local SQLite files is one keystroke.

## Install

```bash
# Cargo
cargo install --locked gobang

# Homebrew
brew install gobang

# verify
gobang --version
```
