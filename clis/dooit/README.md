# dooit

> **A TUI todo manager that treats your task list like a tree, not a
> spreadsheet** — every workspace can hold a hierarchy of nested
> todos, each with due dates, recurrence, urgency, status and free-form
> tags, all editable via vim-style keybinds and rendered in a textual
> (Python `textual` framework) terminal UI. Pinned to **v3.3.4**
> (commit `ba2a7efdbb56297ee7ed9273232cce265e3254da`,
> [LICENSE](https://github.com/dooit-org/dooit/blob/main/LICENSE), MIT).

Source: <https://github.com/dooit-org/dooit>

## TL;DR

Most terminal todo tools force one of two shapes: a flat append-only
list (`todo.txt`, `ttdl`) or a heavyweight kanban (`taskwarrior`
plus a TUI front-end). `dooit` picks the middle: a left pane of
**workspaces** (project / area / context) that themselves nest
arbitrarily, and a right pane of **todos** under the selected
workspace that *also* nest arbitrarily. You navigate with `j/k`,
add a sibling with `a`, a child with `A`, edit fields in place
(`d` → due date, `r` → recurrence cron-ish string, `e` → effort,
`t` → tags), and the whole tree is persisted to a SQLite database
under `$XDG_DATA_HOME/dooit`. Theming, key bindings, and even the
column layout are configured in pure Python (`config.py`),
because dooit imports your config as a module — so a one-line
config change like `dooit.api.theme.background_1 = "#1e1e2e"` is
literally how you re-skin it.

## Install

```bash
# pipx (recommended; isolates the Python env)
pipx install dooit

# pip (user install)
pip install --user dooit

# from source
git clone https://github.com/dooit-org/dooit
cd dooit
pip install -e .
dooit --version    # dooit 3.3.4
```

Run it:

```bash
dooit                       # opens the TUI on the default DB
DOOIT_DB=/tmp/scratch.db dooit   # ephemeral DB for a quick try
```

Inside the UI: `?` shows the cheat sheet, `:q` quits, `Ctrl+s`
forces a save, `/` filters the current pane.

## Why it's worth a slot in the zoo

There is a real gap between "one flat plaintext list" and
"taskwarrior + a calendar plugin". `dooit` fills it cleanly: you
get *structure* (nested workspaces and nested todos) without
giving up the keyboard-only, single-binary, runs-over-SSH
ergonomics that make terminal task managers worth using in the
first place. It is also one of the better real-world demos of the
`textual` framework — if you want to see how a non-trivial
Python TUI is built (reactive widgets, CSS-like styling,
async event handling), the `dooit/ui/` tree is readable and
small. Finally, it is one of the few task managers where the
config language is the host language, so you can do unusual
things (compute today's theme from `datetime.now().hour`, pull
tags from an external API at startup) without patching the
binary.

## Where it sits

- vs `taskwarrior` / `timewarrior`: those are the heavyweight
  reference (UDA, recurring tasks, hooks, sync server, decades of
  reports). `dooit` is the lightweight, opinionated TUI that
  trades extensibility for a cleaner default experience.
- vs `ttdl` / `todo.txt-cli`: those are append-only flat-file
  tools, optimized for grepping and scripting. `dooit` is for
  people who actually want to *navigate* the list.
- vs `dijo` (habit tracker): different shape — `dijo` is a daily
  grid of binary checks; `dooit` is open-ended hierarchical
  todos.
- vs `vit` (Vim-style taskwarrior front-end): same vim feel, but
  `vit` rides on a taskwarrior backend; `dooit` is its own world
  with its own SQLite store.
- vs Notion / Todoist / TickTick: cloud-free, keyboard-first,
  no account, no sync (the SQLite file *is* your sync — put it
  in a git repo or a Syncthing folder).

## Footguns

- **Config is executable Python.** A typo in `config.py` will
  raise on startup and dooit will refuse to launch. Run
  `python -c "import config"` in the config dir before opening
  a fresh terminal.
- **Sync is your problem.** There is no built-in sync. The
  on-disk format is SQLite; two machines editing the same DB
  concurrently via Dropbox/iCloud can corrupt it. Use Syncthing
  with file-versioning, or a git remote you push to manually.
- **3.x broke 2.x configs.** The 3.0 rewrite moved from a YAML
  config to the Python module shown above, and renamed several
  theme tokens. Old `~/.config/dooit/config.yaml` is ignored —
  there is a migration note in the v3.0 release.
- **No native mobile / web client.** "Open the SQLite file on
  your phone" is not a thing. If you need cross-device edits
  outside SSH, this is the wrong tool.
- **Recurring tasks are simple.** `dooit` handles "every N
  days/weeks/months", not arbitrary RFC-5545 RRULEs. For
  complex recurrences (third Tuesday of the month, etc.) reach
  for taskwarrior.
- **The DB lives at `$XDG_DATA_HOME/dooit/dooit.db`** by
  default. Back it up. There is no undo for a deleted workspace
  — confirmation prompt, then it is gone from the tree.
