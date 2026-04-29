# litecli

- **Repo**: https://github.com/dbcli/litecli
- **Version**: v1.17.1
- **License**: BSD-3-Clause ([LICENSE](https://github.com/dbcli/litecli/blob/main/LICENSE))

## What it is

A command-line client for SQLite with auto-completion, syntax highlighting, and a friendly REPL. Sister project to `pgcli` and `mycli` from the dbcli family — same prompt-toolkit foundation, same keystroke muscle memory.

## Why it's in the zoo

The stock `sqlite3` shell is a blank prompt with no completion, no highlighting, and no help discovering tables. litecli reads the schema on connect and then auto-completes table names, column names, SQL keywords, functions, and even values from indexed columns. Multi-line editing, vi/emacs key bindings, named queries (`\f` favourites), output to CSV/TSV/HTML/JSON, and FTS-friendly pagination — it turns a throwaway database into one you'd actually keep open.

## Install

```sh
brew install litecli
# or
pipx install litecli
```

## Quick example

```sh
litecli mydb.sqlite
> SELECT * FROM users WHERE created_at > date('now','-7 day');
```
