# grv

> **GRV — Git Repository Viewer**, a curses-style ncurses TUI for
> browsing a git repository (refs / commits / diffs / status / file
> history) entirely from the keyboard, written in Go. Pinned to
> **v0.3.2**, GPL-3.0 — license file:
> [LICENSE](https://github.com/rgburke/grv/blob/master/LICENSE).

Source: <https://github.com/rgburke/grv>

## TL;DR

`grv` is the *read-and-navigate* TUI for a git repo: a left-side
**ref view** (branches / tags / remotes), a top **commit view** for
the selected ref, and a bottom **diff view** for the selected commit
— all three updating in lockstep as you move the cursor, all three
keyboard-driven (vim-shaped: `j` / `k` / `Ctrl-d` / `Ctrl-u` /
`gg` / `G` / `/pattern`). It also ships a **status view** (the
working-tree changes), a **file view** (any tracked file's git log
with per-line `git blame` annotations), a **summary view**, and a
**grv command bar** (`:` opens a vi-style command line accepting
`:filter`, `:sort`, `:set`, `:theme`, `:q`).

The reason it earns a slot in a catalog already containing
[`tig`](../tig/), [`gitui`](../gitui/), [`lazygit`](../lazygit/),
[`gitu`](../gitu/), [`serie`](../serie/), and [`git-graph`](../git-graph/)
is that grv's identity is **filtering and querying** rather than
**staging and committing**: the command bar lets you express
"commits authored by `alice` whose message matches `/perf/` on the
`release/*` refs" as a `:filter` expression, save it as a custom
view, and bind it to a key in `~/.config/grv/grvrc` — the result is
closer to a *query workbench* over the commit graph than a TUI git
client.

## Install

```bash
# Homebrew (macOS)
brew install grv

# From source (requires Go + libreadline + ncurses + curl)
go install github.com/rgburke/grv@latest

# Pre-built Linux binary (from releases)
curl -L -o grv https://github.com/rgburke/grv/releases/download/v0.3.2/grv_v0.3.2_linux64
chmod +x grv && mv grv ~/.local/bin/
```

## Example commands

```bash
# Open the current repo
cd ~/code/myrepo && grv

# Open a specific repo path
grv -repoFilePath ~/code/myrepo

# Read configuration from an alternate file
grv -configFile ~/.config/grv/release-review.grvrc
```

Inside the TUI:

```
?           help (lists every binding for the current view)
/pattern    search forward in the active view
n / N       next / previous match
:q          quit
:filter author:alice message:/perf/
:sort author
:theme classic
:vsplit DiffView
```

## Niche it occupies

**Query-shaped, read-only git repository TUI** — sits beside `tig`
(the canonical curses git browser; grv adds a richer command-bar
filter language and saved custom views) and orthogonal to
[`lazygit`](../lazygit/) / [`gitui`](../gitui/) / [`gitu`](../gitu/)
which lead with *staging / committing / rebasing* (write-shaped
flows). Pick `grv` when the verb is "investigate the history of
this repo" — code-review prep, blame-driven debugging, release-notes
gathering, "who introduced this regression" sweeps — and you want
the navigation to be one keystroke per move with multiple synced
panes.

## Citation

- Repo: <https://github.com/rgburke/grv>
- Latest release: **v0.3.2**
- License: **GPL-3.0**
- License file: [LICENSE](https://github.com/rgburke/grv/blob/master/LICENSE)
