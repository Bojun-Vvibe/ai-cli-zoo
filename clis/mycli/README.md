# mycli

- **Repo**: https://github.com/dbcli/mycli
- **Version**: v1.70.0
- **License**: BSD-3-Clause (`LICENSE.txt`)

## What it is

A command-line client for MySQL, MariaDB, and Percona with auto-completion and syntax highlighting. Pulls table, column, and keyword completions live from the connected schema.

## Why it's in the zoo

The default `mysql` REPL is a blank prompt with no help. mycli (sibling to pgcli/litecli) gives you smart completion, multi-line editing, and pretty table output without leaving the terminal — the closest thing to a desktop GUI in pure CLI form.

## Install

```sh
brew install mycli
```

## Quick example

```sh
mycli -h 127.0.0.1 -u root mydb
```
