# hishtory

## Overview
`hishtory` is a "better shell history": every command you run is captured with rich context (exit code, runtime, hostname, cwd, git remote), stored in a local SQLite DB, and optionally end-to-end encrypted and synced across machines through a self-hostable server.

## Repo URL
https://github.com/ddworken/hishtory

## Version
`v0.335` (released 2025-02-07)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Why it matters
- Replaces the brittle `~/.bash_history` / `~/.zsh_history` flat files with a queryable store — `hishtory query 'kubectl apply'` returns matches across every host you use.
- Captures fields plain history cannot: exit code, duration, working directory, hostname, current git project. Lets you ask "what failing commands did I run yesterday in this repo?".
- Sync layer is opt-in and end-to-end encrypted; keys never leave the device. You can self-host the sync server.
- ~3k stars, packaged for brew/apt, supports bash / zsh / fish.

## Install
```sh
# Official one-liner installer (downloads release binary, sets up shell hook)
curl https://hishtory.dev/install.py | python3 -

# macOS via brew
brew install hishtory

# Source: https://github.com/ddworken/hishtory/releases
```

## Quick example
```sh
# Query across every machine where hishtory is installed
hishtory query kubectl apply

# Last 20 failing commands in the current repo
hishtory query exit_code:1 cwd:$PWD

# Disable sync and stay 100% local
hishtory disable-sharing
```

## Comparison context in this zoo
- Adjacent to existing shell-history / prompt entries:
  - `atuin` — closest peer; magic up-arrow search + sync. Atuin emphasizes interactive search; hishtory emphasizes structured query and cross-host context.
  - `mcfly` — neural-network ranked Ctrl-R replacement, single-host only, no sync.
  - `starship` — prompt renderer, complementary (display) vs. hishtory (storage/query).
- Pick `hishtory` when you want commands as structured records you can `query` like a log; pick `atuin` when you want the slickest Ctrl-R UX.
