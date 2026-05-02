# ttdl

> **Terminal todo.txt manager that treats the
> [todo.txt format](http://todotxt.org/) as a real, queryable
> data store** — every task is a single line, every line is
> grep-able, and the CLI layers tag-aware filtering, recurrence,
> time tracking, and customizable column rendering on top of
> that one plain-text file. Pinned to **v6.1.1**
> ([LICENSE](https://github.com/VladimirMarkelov/ttdl/blob/master/LICENSE),
> MIT).

Source: <https://github.com/VladimirMarkelov/ttdl>

## TL;DR

`ttdl` (Terminal ToDo List) is a single static Rust binary
that operates on a `todo.txt` file in your home (or any path
you point it at) and gives you the full set of operations a
modern task manager needs without ever leaving the format:
`add`, `done`, `undone`, `edit`, `rm`, `archive` (move
completed lines to `done.txt`), plus rich `list` queries
that read like a small DSL. The DSL is the value: filters by
project (`+proj`), context (`@ctx`), tag (`tag:value`),
priority (`--pri=A+` for "A or higher"), recurrence
(`--rec=any`), and date ranges with relative literals
(`--due ..today` for overdue + due-today, `--due -2d..2d`
for "slightly overdue or due within 2 days", `--due
-first..last` for "this month"). Recurrence is first-class
and supports both **strict** (`rec:+1m` — next due = current
due + interval, good for fixed billing dates) and
**non-strict** (`rec:1m` — next due = today + interval, good
for "mow the lawn at least monthly") modes, including
end-of-month edge handling and an `until:` tag that
auto-stops the chain. Column rendering is fully customizable
via `--fields`, `--auto-hide-cols`, `--auto-show-cols`, and
`--clean-subject`, so you can render the same data as a
compact one-liner-per-task list, a wide table with every
custom tag broken out as its own column, or anything in
between. There's also a built-in time tracker (`start` /
`stop` per task), a `--human` flag that turns absolute dates
into `in 2d` / `6d ago`, and an interactive editor mode
that hands a task off to `$EDITOR` and re-parses the result.

## Install

```bash
# Homebrew (macOS / Linux)
brew install ttdl

# Cargo (any platform with Rust 1.31+)
cargo install ttdl

# Cargo, no markdown rendering
cargo install ttdl --no-default-features

# Arch (AUR)
paru -S ttdl

# Scoop (Windows)
scoop bucket add extras
scoop install ttdl

# Single-binary download (GitHub releases, v6.1.1)
curl -L -o ttdl.tar.gz \
  https://github.com/VladimirMarkelov/ttdl/releases/download/v6.1.1/ttdl_6.1.1_linux-x64-musl.tar.gz
tar xzf ttdl.tar.gz && sudo mv ttdl /usr/local/bin/

# Build from source pinned to release tag
git clone --depth 1 --branch v6.1.1 https://github.com/VladimirMarkelov/ttdl.git
cd ttdl && cargo install --locked --path .
```

`ttdl` reads its config from `$XDG_CONFIG_HOME/ttdl/ttdl.toml`
(or the OS-specific equivalent) and the path to the todo
file from the config's `filename`, the env var
`TTDL_FILENAME`, the CLI flag `--todo-file`, or `--local`
(forces `./todo.txt`). Bootstrap a default config with
`ttdl --init` (user dir) or `ttdl --init-local` (cwd).

## Usage

```bash
# Add tasks. Priority is `(A)`-`(Z)` and must be the first token.
ttdl add "(A) ship the v1 release +work @repo due:2026-05-10"
ttdl add "pay credit card +finance due:2026-05-29 rec:+1m"      # strict monthly
ttdl add "mow lawn +home due:2026-05-10 rec:1m"                  # non-strict monthly
ttdl add "water plants +home due:1w t:today rec:1w until:1m"     # date-expression tags

# List + filter
ttdl list                              # all open tasks, default columns
ttdl list +work @repo                  # in project work AND context repo
ttdl list --pri=B+                     # priority B or higher
ttdl list --due ..today                # overdue + due today
ttdl list --due -first..last           # due any day this month
ttdl list --completed -1w.. -a         # finished within the last 7 days
ttdl list -e "^fix.*api$"              # regex over subject/projects/contexts
ttdl list --filter "due=..today;pri=A..C" --filter "+home"  # arbitrary AND/OR

# Mark done / undone (recurrent tasks auto-create the next instance)
ttdl done 5
ttdl done 3 -r "shipped in v1.2"        # append a resolution note on completion
ttdl undone 5

# Edit / append / prepend / set tags
ttdl edit 7 --set-due=today+1w --set-pri=A
ttdl append 7 "(see #1234)"
ttdl edit 7 --set-tag rec:1w

# Maintenance
ttdl archive                           # move completed lines to done.txt
ttdl stats                             # counts by project / context / status
ttdl calendar 2w                       # 2-week calendar view of due dates
ttdl agenda 7                          # next 7 days, grouped by day
```

## Why it's interesting

The `todo.txt` slot has a lot of clients (the original
Python `todo.sh`, GUI apps, mobile apps), but most either
treat the file as opaque storage and re-add their own DB
on top, or stay strictly on the literal format and force
you to grep manually. `ttdl` is the rare one that
**respects the file as the source of truth** — every
operation rewrites `todo.txt` in canonical form and you can
hand-edit it between commands without breaking anything —
while still giving you the query power of a tag-aware DB.
Pick `ttdl` when (a) you've outgrown
[`taskwarrior`](../taskwarrior/) / [`taskwarrior-tui`](../taskwarrior-tui/)
because their SQLite-backed model means you can't `cat`,
`sed`, or sync the data with plain `git`/`rsync` /
`syncthing` and you want a file you can paste into a wiki,
(b) you want recurrence with both strict and non-strict
semantics in one tool (most todo.txt clients pick one), or
(c) you live in a [todo.txt format](http://todotxt.org/)
ecosystem already (e.g. you sync via `syncthing` to
[Simpletask](https://github.com/mpcjanssen/simpletask-android)
on Android) and want a desktop client with first-class
filtering and column control. Not the right pick when you
need a server with multi-user accounts (use
[`vikunja`](../vikunja/) or hosted), when you want a Kanban
board UI (use [`kanban-tui`](../kanban-tui/) or a desktop
app), or when your team has standardized on a non-todo.txt
format (Org-mode, Markdown checklists, Jira). The author
has been maintaining `ttdl` since 2018 with a clean
`changelog`, monthly-ish releases, and a deliberate "no new
file format" stance — every feature has to fit inside a
todo.txt line via existing tag conventions, which is a
rare and valuable design constraint. Compare with
[`todoman`](../todoman/) (CalDAV/iCalendar-backed, good if
you want sync via a CalDAV server) and
[`dstask`](../dstask/) (git-backed YAML files, good for
team-shared task lists in a repo) — `ttdl` occupies the
"single-file, plain-text, scriptable, recurrence-aware"
slot they don't.
