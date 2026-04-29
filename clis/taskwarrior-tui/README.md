# taskwarrior-tui

> Snapshot date: 2026-04. Upstream: <https://github.com/kdheepak/taskwarrior-tui>

A Rust TUI front-end for [Taskwarrior](https://taskwarrior.org/) —
the long-lived, plain-text, plain-`~/.task/`-on-disk task manager.
Renders your task list with vim-style navigation, live filtering,
inline edits, calendar view, dependency graph, and keyboard-only
add/modify/annotate flows, while delegating every state change back
to the `task` binary so the database stays compatible with every
other Taskwarrior client (mobile, web, sync server).

## 1. Install

- `brew install taskwarrior-tui` — Homebrew (macOS / Linux); pulls
  `task` (Taskwarrior) as a dependency
- `cargo install taskwarrior-tui --locked` — from crates.io
- `pacman -S taskwarrior-tui`, `nix-env -iA nixpkgs.taskwarrior-tui`
- Pre-built binaries on the
  [releases page](https://github.com/kdheepak/taskwarrior-tui/releases)
- Requires `task` >= 2.6.0 on `$PATH` and an initialised
  `~/.taskrc` (`task` will prompt to create one on first run).

## 2. Version pin

**v0.27.0**. Verify with `taskwarrior-tui --version`.

## 3. License

MIT. SPDX: `MIT`. Full text at
[`LICENSE`](https://github.com/kdheepak/taskwarrior-tui/blob/main/LICENSE)
in the repository root.

## 4. What it does

- Three-pane layout: task table on the left (configurable columns:
  ID, project, tags, urgency, due, description), preview pane on
  the right with the full task JSON, and a context bar at the
  bottom showing the active Taskwarrior context / report.
- Vim bindings: `j/k` move, `/` filter, `n/N` next/prev match, `:`
  command line that proxies to `task`, `q` quit.
- Inline verbs: `a` add, `m` modify, `e` edit (opens `$EDITOR` on
  the task), `d` mark done, `x` delete, `u` undo (proxies
  `task undo`), `s` start/stop a timer, `A` annotate.
- Calendar view (`c`), dependency tree (`g`), and project breakdown
  (`p`); each rendered from `task` reports so they pick up your
  existing custom report definitions in `~/.taskrc`.
- Live refresh — file-system watch on `~/.task/` means external
  edits (mobile sync, `taskd`, scripts) appear without a manual
  reload.
- Configurable via `~/.config/taskwarrior-tui/config.toml`: column
  set, key bindings, colour palette, default report (`next`,
  `ready`, `recurring`, your custom report).

## 5. Install & first use

```bash
brew install task taskwarrior-tui
task add "draft release notes for next mobile build" \
  project:release due:friday +writing
taskwarrior-tui                       # opens on the default report
# inside the TUI:
#   /writing    filter to tasks tagged +writing
#   enter       open the highlighted task in the preview
#   m           modify (e.g. change due date)
#   d           mark done; row strikes through and refresh fires
#   c           calendar view; ESC back
#   q           quit
```

## 6. Example: weekly review loop

```bash
# 1. open the "ready" report (filtered, sorted by urgency)
taskwarrior-tui --report ready

# 2. inside the TUI, walk top-to-bottom:
#    - press `e` to edit any task whose description is stale
#    - press `m` then type `due:next-friday` to reschedule
#    - press `s` on the task you'll work next to start the timer
# 3. quit with `q`; the underlying ~/.task store is now updated
#    and any taskd / inthe.am / mobile client picks up the changes.

# scripted complement (no TUI):
task export ready | jq '.[] | select(.urgency > 5) | .description'
```

## 7. Why it matters in this catalog

Taskwarrior is the long-running, plain-text counterpart to the
project-management SaaS tools that AI agent CLIs increasingly
integrate with. When an agent loop wants to record "follow up on
the failing test" or "rebase this branch tomorrow" without reaching
for a server-backed system, `task add` is one subprocess call and
the result is grep-able forever. `taskwarrior-tui` is the human
review surface on top — a place to triage what the agent enqueued,
mark items done, and re-prioritise — without leaving the terminal
or learning the ~40 `task` subcommands by heart. It pairs
particularly well with [`atuin`](../atuin/) (history) and
[`hishtory`](../hishtory/) (cross-host history) for "what was I
doing yesterday on the other machine" reconstruction.

## 8. Alternatives

- `task` directly — always available, scripting-friendly, but the
  mental model of `task <filter> <command> <modifications>` is
  steep for casual triage. `taskwarrior-tui` wraps it.
- `vit` — older curses TUI for Taskwarrior, Python-based, slower
  refresh; mostly superseded.
- [`dijo`](https://github.com/NerdyPepper/dijo) — habit tracker
  TUI, complementary rather than overlapping.
- Org-mode in Emacs — different storage, different ecosystem;
  pick one and stick with it.
- Linear / Jira / Things 3 / OmniFocus — server-backed; choose
  Taskwarrior + this TUI when you want plain text, local-first,
  zero-subscription, and full grep + jq tooling on the data.

## 9. Telemetry

No analytics, no outbound network. The TUI only reads `~/.task/`
and execs `task`; if you've configured Taskwarrior with `taskd`
sync, that traffic is unchanged by the presence of the TUI. Safe
on air-gapped machines.
