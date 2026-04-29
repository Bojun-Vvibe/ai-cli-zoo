# dijo

> **A scriptable, curses-based, digital habit
> tracker** — a tiny Rust TUI that renders the
> current month as a grid of habits × days, lets you
> mark each cell with a single keypress, and stores
> everything as a plain-text file you can sync via
> any mechanism (`git`, Syncthing, Dropbox, scp).
> Pinned to **v0.2.7**
> ([LICENSE](https://github.com/oppiliappan/dijo/blob/master/LICENSE),
> MIT).

Source: <https://github.com/oppiliappan/dijo>

## TL;DR

`dijo` ("dee-jo", Tamil for "habit") is the
terminal-native answer to the wave of subscription
habit-tracker apps. The whole UI is one screen: a
left column listing habits, a horizontal row of
days for the current month, and a cell at each
intersection that is empty, ticked (✓), or skipped.
You move the cursor with `hjkl` / arrow keys, hit
space to toggle today's cell, and the current
streak / monthly completion percentage updates
live in the right margin. Habits come in three
flavours — *bit* (binary did/didn't, e.g.
"meditate"), *count* (numeric goal per day, e.g.
"pages read ≥ 20"), and *float* (decimal goal per
day, e.g. "litres of water ≥ 2.0") — defined in a
single TOML config (`~/.config/dijo/config.toml`)
that you can hand-edit or generate from the
in-app `:add` / `:delete` / `:mprev` / `:mnext`
commands. State is persisted as a plain JSON file
(`~/.local/share/dijo/habits.json`) you can grep,
diff, and back up, which means a `git` repo on
your dotfiles directory is a perfectly serviceable
sync layer with full history. The binary is ~2 MB,
zero runtime deps, starts in <50 ms.

## Install

```bash
# Cargo (any platform with a Rust toolchain)
cargo install dijo

# Arch (AUR)
yay -S dijo

# Nix
nix-env -iA nixpkgs.dijo

# Homebrew tap (community)
brew install dijo

# pre-built binary
curl -L https://github.com/oppiliappan/dijo/releases/latest/download/dijo-x86_64-linux \
  -o /usr/local/bin/dijo && chmod +x /usr/local/bin/dijo

# verify
dijo --version    # dijo 0.2.7
```

## Basic usage

```bash
# launch the TUI; first run creates the config + state files
dijo

# inside the TUI:
#   space            toggle today's cell on the focused habit
#   h j k l          move cursor (vim-style)
#   ← → ↑ ↓          move cursor (arrows)
#   v                switch to "view-only" focus mode
#   w                toggle weekly view
#   :add <name>      add a binary habit
#   :addc <name> N   add a count habit with daily goal N
#   :addf <name> X   add a float habit with daily goal X
#   :delete <name>   remove a habit
#   :mnext / :mprev  next / previous month
#   :w               write to disk now (also done on quit)
#   :q               quit
#   ?                keybinding help

# config lives at ~/.config/dijo/config.toml — keys: look, dimensions,
# colorscheme, future_chars, etc.

# state lives at ~/.local/share/dijo/habits.json — JSON array, one
# entry per habit, with per-day records. Plain text, diffable,
# git-friendly:
git -C ~/.local/share/dijo init && git add habits.json && \
  git commit -m "$(date +%F)"

# scripted bulk import — write JSON directly, dijo will pick it up
jq '. += [{"name":"floss","kind":"bit","stats":{}}]' \
  ~/.local/share/dijo/habits.json > /tmp/h.json && \
  mv /tmp/h.json ~/.local/share/dijo/habits.json
```

## When to choose

- **You already live in the terminal and want
  habit-tracking that doesn't pull you out of it**
  — `dijo` opens in a tmux pane, draws a grid, and
  closes. No browser, no electron, no notifications,
  no account.
- **You want full ownership of your data** — the
  state file is a single JSON document on your own
  disk. Back it up the way you back up anything
  else; sync via `git` for free history; export to
  CSV with `jq` in one line.
- **You like the GitHub-contribution-graph
  aesthetic for personal goals** — the `dijo`
  monthly grid is essentially that pattern, with one
  row per habit, and the visual feedback is the
  whole motivation loop.
- **You want a tiny, scriptable building block, not
  a product** — the JSON state and TOML config mean
  you can wrap `dijo` in cron jobs, post-commit
  hooks, Home Assistant automations, or `gum`-style
  shell scripts that mark cells based on external
  signals (`git log`-derived "shipped a commit
  today", fitness-watch CSVs, etc.).

## When NOT to choose

- **You want push notifications, reminders, or
  smartphone access** — `dijo` is a desktop TUI with
  no daemon, no mobile client, no nag system. Use
  Habitica, Streaks, Loop Habit Tracker (Android),
  or any of the SaaS options if those matter.
- **You need rich analytics** — `dijo` shows
  current streak and monthly completion percentage.
  For per-week heatmaps, correlations between
  habits, or multi-year trends, export the JSON and
  plot it yourself, or pick a heavier tool.
- **You want a polished GUI** — by design the UI is
  curses, monospace, and minimal. People who want a
  designed product won't enjoy it.
- **You need multi-user / team tracking** — `dijo`
  is single-user, single-file. There is no shared
  habit / accountability-partner mode.

## Why it fits the zoo

The zoo's "personal-productivity TUI" cluster
([`taskwarrior`](../taskwarrior/),
[`vit`](../vit/),
[`tasklite`](../tasklite/),
[`harvest`](../harvest/),
[`watson`](../watson/),
[`timewarrior`](../timewarrior/)) collects
single-purpose terminal tools that own one slice of
self-management with a plain-text data model.
`dijo` slots into that group as the *habits*
counterpart: it is to habit tracking what
`taskwarrior` is to todos and `timewarrior` is to
time. All three share the same shape — local
state file, scriptable from the shell, optional
sync via whatever you already use, no cloud
account — and stack cleanly together in a tmux
session.

## Upstream pointers

- Repo: <https://github.com/oppiliappan/dijo>
- Release notes: <https://github.com/oppiliappan/dijo/releases>
- License: [MIT](https://github.com/oppiliappan/dijo/blob/master/LICENSE)
- Wiki / config schema: <https://github.com/oppiliappan/dijo/wiki>
- Author: [@oppiliappan](https://github.com/oppiliappan)
  (also [`gtm`](https://github.com/oppiliappan/gtm),
  a tiny git-time-tracker, and other minimalist
  TUIs).
