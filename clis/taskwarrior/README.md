# taskwarrior

- **Repo:** https://github.com/GothenburgBitFactory/taskwarrior
- **Version:** v3.4.1 (current stable on the v3.x line; v3 introduced the SQLite-backed replica + native sync server)
- **License:** MIT — see [`LICENSE`](https://github.com/GothenburgBitFactory/taskwarrior/blob/develop/LICENSE)
- **Language:** C++ + Rust (C++ command surface and DOM/expression engine; the v3 storage + sync layer is the Rust [`taskchampion`](https://github.com/GothenburgBitFactory/taskchampion) library, embedded as a static lib; SQLite as the on-disk format)
- **Install:** `brew install task` · `apt install taskwarrior` · `dnf install taskwarrior` · `pacman -S taskwarrior` · or build from source (`cmake -S . -B build && cmake --build build`); the binary is `task` and pairs with the optional `taskwarrior-sync-server` (Rust) for self-hosted sync, or with a Taskchampion-compatible cloud (AWS S3, GCP GCS, local file replica)

## Overview

`taskwarrior` is a command-line task manager that
treats the task list as a **typed, queryable database
with first-class filters, scripts, and sync** rather
than a Markdown checklist. Every task has structured
attributes (`description`, `project`, `tags`,
`priority`, `due`, `wait`, `scheduled`, `recur`,
`depends`, `annotations`, plus arbitrary user-defined
attributes via `uda.*` config), and the CLI is a
filter-then-action grammar: `task project:home and
+next and due.before:eow list` reads as "from tasks in
the `home` project, tagged `+next`, due before end of
week, list them." The same filter language drives
every command — `list`, `count`, `done`, `modify`,
`annotate`, `delete`, `summary`, `burndown.daily`,
`history.monthly` — so once you know the filter
syntax you know how to ask any question of the data.
v3 replaced the legacy flat-file format with
**Taskchampion**, a Rust replica + sync library that
gives you a real SQLite database under
`~/.local/share/task/taskchampion.sqlite3`, atomic
writes, and a CRDT-style sync protocol against
either a self-hosted `taskwarrior-sync-server`, an
S3 / GCS bucket, or a local file replica — multi-
device sync without a third-party SaaS, end-to-end
encrypted with a client-side key. Recurring tasks
(`recur:weekly due:fri`), deferred / waiting tasks
(`wait:2026-05-15`), task dependencies (`depends:42`
which suppresses the dependent until 42 is done),
and per-context filters (`task context define work
'project:work'` then `task context work` activates
the filter implicitly for every command) compose into
a workflow that scales from 50 personal todos to a
few thousand tasks across many contexts. The
`hooks/` directory under `~/.config/task/` runs
scripts on `on-add` / `on-modify` / `on-launch` /
`on-exit` events, so external integrations (post a
Slack message when a task tagged `+blocking` is
added, sync a `+calendar` task into iCal, push
metrics into Prometheus textfile exporter) are just
shell scripts in the right directory.

## Niche

**The structured, filter-first, sync-capable task
database for the command line — typed attributes
(`project`, `tags`, `due`, `wait`, `recur`,
`depends`, plus UDAs), a uniform filter grammar
across every verb, recurring + deferred + dependent
tasks, multi-device sync via a self-hosted server or
S3 / GCS with end-to-end encryption, and a hook
system that lets shell scripts react to every
add / modify / done event.** The role is "the task
manager you reach for when a Markdown TODO list has
stopped scaling, you want to query 'what's due
this week, untagged, in any project but @home' as a
one-line CLI invocation, and you want sync across
your laptop / desktop / server without trusting a
SaaS vendor with the data." Competing universe:
todo.txt / dstask / topydo / org-mode / TickTick /
Todoist. See comparisons below.

## When to use

- You have **more than 50 active tasks** across
  multiple projects and contexts and need to query /
  filter them by combinations of attributes:
  `task project:work and +urgent and due.before:friday`.
- You want **recurring tasks** with proper instance
  semantics: `task add "weekly review" recur:weekly
  due:fri` materialises a new instance each week,
  marking one done does not affect the others.
- You want **task dependencies** that hide the
  dependent until the blocker is done: `task add
  "deploy v2" depends:127` keeps `deploy v2` out of
  the `next` list until #127 is closed.
- You want **deferred tasks** that disappear from
  the active list until a date: `task add "tax
  return" wait:2026-04-01` shows it only after April
  1st.
- You want **multi-device sync** without a SaaS:
  `task sync` against your own
  `taskwarrior-sync-server` on a $5 VPS, or against
  an S3 / GCS bucket, end-to-end encrypted with a
  client-side key.
- You want **per-context filters** so the same
  command shows different views from your work
  laptop vs personal laptop: `task context work`
  implicitly scopes every subsequent command.
- You want **a hook system** for integrations
  (Slack on add, iCal sync, Prometheus metrics,
  desktop notification on due-soon).
- You want **structured exports** for downstream
  tools (LLM agents, dashboards, time-tracking):
  `task export +urgent` emits one JSON object per
  task with all attributes.

## When NOT to use

- You want **the simplest possible plain-text TODO
  list** that lives in a Markdown / `todo.txt` file
  you can grep, sync via git, and edit in any
  editor — pick `todo.txt` (not in zoo) or
  `dstask` (not in zoo). Taskwarrior's database is
  not the file you `cat ~/TODO.md` — it's a SQLite
  blob. Pick text-first when grep / git is the
  workflow.
- You want **rich, hyperlinked, hierarchical notes
  with tasks embedded** rather than tasks as the
  primary unit — pick Emacs `org-mode` (not in
  zoo) or [`obsidian-cli`](../obsidian-cli/) /
  [`logseq`](../logseq/) (if added). Org's superpower
  is "documents that contain tasks"; taskwarrior's
  is "tasks that contain documents."
- You want **a GUI / mobile app** with great default
  UX and don't mind a SaaS — TickTick / Todoist /
  Things 3 are better at that. Taskwarrior's GUIs
  are third-party and uneven; the value is in the
  CLI + filter language.
- You only ever want to **track a flat shopping
  list** with no projects, tags, due dates, or
  filters — `echo bread >> ~/list && cat ~/list`
  is the right tool.
- You need **real-time multi-user collaboration**
  on the same task list (multiple people editing
  simultaneously with presence) — taskwarrior sync
  is eventually-consistent CRDT, fine for one human
  across N devices, not designed for "team kanban
  with live cursors."

## Comparison vs alternatives in zoo

- [`khal`](../khal/) (if added) — CLI calendar.
  Complementary — taskwarrior tracks "things I have
  to do at some point", khal tracks "things that
  happen at a specific time"; export taskwarrior's
  due-dated tasks into iCal via a hook to surface
  them in khal.
- [`buku`](../buku/) (if added) — bookmark / notes
  database. Orthogonal — buku is to URLs what
  taskwarrior is to tasks; same CLI-as-database
  philosophy, different domain.
- [`jrnl`](../jrnl/) (if added) — append-only
  journal. Complementary — jrnl for "what
  happened", taskwarrior for "what needs to
  happen".
- [`chezmoi`](../chezmoi/) — dotfile manager.
  Pair with taskwarrior to ship `~/.config/task/`
  + hooks across machines so a fresh box gets the
  same filters and contexts after one
  `chezmoi apply`.
- [`atuin`](../atuin/) — shell-history sync. Same
  end-to-end-encrypted-sync mental model
  (server-side blob, client-side key); a natural
  pair on a fresh laptop set-up checklist.
- [`himalaya`](../himalaya/) — CLI mail client.
  Complementary — a hook on `+followup` tagged
  taskwarrior tasks can `himalaya send` a
  pre-canned email, closing the loop between
  inbox-zero workflows and task-list workflows.

## Why it earns a slot in an AI-native workflow

Most LLM-CLI agent stacks accumulate "things the
agent surfaced that I should do later" — a flaky
test it found, a TODO it noticed in code review, a
follow-up question it could not resolve, a paper
it suggested I read, a refactor it deferred to the
human. A Markdown checklist works for the first
30; past that, the human can no longer answer
"what did the agent flag this week, in this
project, that's still open?" without grepping
the chat history. Taskwarrior with a hook that
ingests an agent's `flag-for-followup` tool call
(one POST → `task add` with `project:` and
`+from-agent` tag and `annotate:<chat-url>`)
turns that ephemera into a queryable, sync'd,
filter-friendly database the human can drain
deterministically. `task export +from-agent and
status:pending` then becomes the structured input
the *next* agent run reads back as "here's what
the human still owes from previous sessions",
closing the agent ↔ human ↔ agent loop with
something more durable than a chat log. The
filter grammar is also exactly the surface area
an agent finds easy to emit ("show me everything
project:vvibe-cli-zoo and due.before:eow that I
can knock out tonight") without having to learn a
custom DSL — it's regular English-shaped predicate
logic.

## Example invocations

```bash
# Add a task with project, tag, due date
task add "review spec" project:work +urgent due:friday

# List with filter — every verb takes the same filter grammar
task project:work and +urgent and due.before:eow list

# Recurring task (weekly Friday)
task add "team weekly review" recur:weekly due:friday

# Dependent task — hidden until #127 is done
task add "deploy v2" depends:127

# Deferred task — invisible until April 1st
task add "file taxes" wait:2026-04-01

# Mark done by id
task 42 done

# Annotate (attach a note / URL / context)
task 42 annotate "see https://example.com/incident-9912"

# Per-context implicit filter
task context define work 'project:work'
task context work
task list   # implicitly scoped to project:work

# Burndown chart (60-day window)
task burndown.daily

# Structured JSON export — feed to scripts / agents / dashboards
task export +urgent and status:pending | jq '.[] | {id, description, due}'

# Sync (against self-hosted server or S3 / GCS, end-to-end encrypted)
task sync init           # one-time
task sync                # subsequent

# Hook example: ~/.config/task/hooks/on-add.notify
#!/bin/sh
read -r json
desc=$(echo "$json" | jq -r .description)
notify-send "task added" "$desc"
echo "$json"             # must echo back the (possibly modified) task

# UDA — define a custom attribute
task config uda.estimate.type duration
task config uda.estimate.label Estimate
task add "design doc" estimate:2h
task estimate.over:1h list
```
