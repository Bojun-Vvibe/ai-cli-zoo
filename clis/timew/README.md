# timew

- **Repo:** https://github.com/GothenburgBitFactory/timewarrior
- **Version:** v1.9.1 (2025-08-27)
- **License:** MIT ([COPYING](https://github.com/GothenburgBitFactory/timewarrior/blob/develop/COPYING))
- **Language:** C++17 (single static `timew` binary; depends on a C++ stdlib only)
- **Install:** `brew install timewarrior` · `apt install timewarrior` · `dnf install timewarrior` · build from source via CMake (`cmake -S . -B build && cmake --build build && sudo cmake --install build`); also packaged on most BSDs and `pkgsrc`

## What it does

`timew` (Timewarrior) is a **command-line time tracker** by the same group that ships [`taskwarrior`](../taskwarrior/) (GothenburgBitFactory). It records intervals of work — each interval is a start time, an optional end time, and a set of tags — and answers questions about them: how many hours did I spend on `client:acme` this week, what's the daily breakdown across `project:invoice-rewrite` and `meetings`, what was I doing between 14:00 and 16:00 yesterday, did I exceed the 8h/day cap on Tuesday. The surface is minimal and learnable in 10 minutes: `timew start <tags...>` opens a new interval (`timew start client:acme api refactor` starts an interval tagged with three tags); `timew stop` closes the currently-open interval; `timew continue` resumes the most-recently-stopped interval with its tags (one-keystroke "back to work after lunch"); `timew track <range> <tags...>` retroactively records an interval (`timew track 9am - 11:30am client:acme retro`); `timew summary :week` prints the per-day breakdown for the current week; `timew day` / `timew week` / `timew month` open per-period reports; `timew tags` lists every tag ever used; `timew get dom.active.duration` exposes structured DOM-style queries scriptable from `bash`. The data store is plain text under `~/.timewarrior/data/<YYYY>-<MM>.data` — one interval per line, machine-parseable, `git`-trackable, and editable with `$EDITOR` via `timew undo` (last operation) or direct file edit if the live undo stack is not enough. Tags are free-form strings (no schema) but a colon-delimited convention emerges naturally (`client:acme`, `project:billing-v3`, `meeting`, `dnd`, `interrupted`) and `timew summary` aggregates by any tag prefix on demand. Reports are powered by an embeddable extension system: scripts under `~/.timewarrior/extensions/` (Python, shell, Perl — anything with a shebang) read the JSON-shaped intervals stamped on stdin and emit any report you want; the bundled `csv.py`, `totals.py`, and `gantt.py` extensions cover most needs out of the box and `timew report <name> [args...]` invokes them. The killer integration is the `on-modify` hook bundled with [`taskwarrior`](../taskwarrior/): `cp /usr/share/doc/timewarrior/ext/on-modify.timewarrior ~/.task/hooks/` makes `task <id> start` automatically open a `timew` interval with the task's project and tags applied, and `task <id> stop` closes it — Taskwarrior becomes the "what" and Timewarrior becomes the "when", with no double-entry.

## When to pick it / when not to

Pick `timew` when time tracking needs to be **friction-free, local, scriptable, and yours** — freelancers billing by the hour against multiple clients, consultants who need defensible weekly invoices, engineers who want honest data on where the day actually goes, anyone running a personal productivity system who already lives in a terminal and wants tracking that does not require a SaaS account. Concrete cases: a freelance dev billing 4 clients at hourly rates who runs `timew start client:acme feature-X` at the start of each session and `timew summary :week :ids client:` on Friday to produce hours-per-client for invoicing (export via the CSV extension into the billing tool); a Taskwarrior user who wires the `on-modify` hook so every `task 42 start` automatically opens a tagged Timewarrior interval and `task 42 done` closes it, eliminating the "what was I doing in that 90-minute block" question entirely; a remote worker whose timezone varies and who needs the tracker to handle DST and travel correctly (Timewarrior stores everything in UTC and renders in `$TZ`, so a flight from PST to CET does not corrupt the daily summary); a team standardizing on a per-repo `~/.timewarrior/extensions/team-summary.py` that reads intervals from `STDIN` and emits a Slack-shaped weekly digest; a privacy-sensitive contractor whose client contract forbids sending billable hours to a third-party SaaS — Timewarrior's data file never leaves the machine. Pair with [`taskwarrior`](../taskwarrior/) for the task layer above the time layer; pair with [`vd`](../visidata/) / [`miller`](../miller/) / [`xsv`](../xsv/) on the exported CSV for ad-hoc analysis; pair with `git` (private repo) on `~/.timewarrior/` for cross-machine sync — the data files are plain text and merge cleanly under typical solo workflows.

Skip `timew` when the team needs **shared / web-based / role-based time tracking** — a 50-person consultancy where the office manager runs reports across staff needs Toggl / Harvest / Clockify / a real billing system, not a per-laptop CLI. Skip when the workflow is fundamentally GUI / mobile (you start and stop work by tapping a phone widget) — Timewarrior is terminal-first; the mobile story is "SSH into your laptop", which is not a workflow. Skip when accounting integration matters more than tracking accuracy — Timewarrior exports CSV / JSON cleanly but you do most of the integration work yourself; SaaS trackers ship Stripe / QuickBooks / Xero / Wave connectors that take an afternoon to wire vs an afternoon plus ongoing maintenance with `timew + cron + curl`. Skip if you genuinely will not start the timer; the tool's correctness depends on the human running `timew start` / `timew stop` consistently. (The Taskwarrior hook helps a lot, but not infinitely.)

## Vs already cataloged

- **Vs [`taskwarrior`](../taskwarrior/):** complementary, same maintainers. Taskwarrior tracks *what* needs doing (priorities, dependencies, due dates, projects); Timewarrior tracks *when* you actually did it. The bundled `on-modify.timewarrior` Taskwarrior hook makes them act as one tool — `task 42 start` opens a tagged Timewarrior interval, `task 42 stop` closes it. The pair is the canonical Unix-philosophy time-and-task setup.
- **Vs [`watson`](https://github.com/jazzband/Watson):** the closest peer in feature space — both are CLI time trackers with `start` / `stop` / `tag` / `report` / data-as-text. Watson is Python (slower startup, easier to extend in Python directly), syncs to a hosted "Crick" backend optionally, has a built-in projects model with frames-as-Python-pickles. Timewarrior is C++ (fast startup), has no built-in sync (use `git` on `~/.timewarrior/`), uses tags rather than projects (more flexible, less opinionated), and integrates natively with Taskwarrior. Pick Timewarrior when Taskwarrior is already adopted; pick Watson when you want a small Python codebase to extend.
- **Vs [`ttdl`](../ttdl/):** orthogonal — `ttdl` is a todo.txt CLI (task lifecycle), not a time tracker. Pair them: ttdl tracks what's open, timew tracks how long you spent on each.
- **Vs [`dstask`](../dstask/):** orthogonal — `dstask` is a single-binary git-backed task tracker (markdown notes per task); it has no time-tracking surface. Some users pair `dstask` for tasks + `timew` for time, mediated by a small shell hook.
- **Vs [`harlequin`](../harlequin/) / [`pgcli`](../pgcli/) / [`mycli`](../mycli/):** unrelated category — those are SQL TUIs.
- **Vs SaaS time trackers (Toggl, Harvest, Clockify, RescueTime):** different value prop. SaaS trackers are multi-user, web-dashboard, mobile-first, and integrate with billing systems out of the box; Timewarrior is single-user, terminal-only, and integrates with whatever you script. The data-ownership / no-vendor-lockin / works-offline / works-on-an-airplane axes belong to Timewarrior; the team-reporting / pretty-charts / managed-backups axes belong to SaaS.

## Caveats

- **You have to actually start the timer.** No automatic application-window detection, no idle-detection, no "you were idle for 15 min, discount the interval?" prompt. The Taskwarrior hook helps because `task start` becomes the prompt; without it, Timewarrior is as accurate as your discipline. For passive tracking, look at RescueTime or `arbtt`.
- **Tags are free-form strings.** Typos compound: `client:acme` and `client:Acme` are different tags and aggregate separately in `summary`. Adopt a tag convention early (`client:lowercase-slug`, `project:lowercase-slug`, `meeting`, `interrupted`) and stick to it; the `timew tags` command surfaces drift.
- **Single-user, no built-in sync.** The `~/.timewarrior/data/` files are plain text and `git`-friendly, but multi-machine setups need a sync story (private git repo + `git pull` / `git push` is the common pattern; restic / syncthing also work). Concurrent edits on two machines without sync between them produce a merge conflict you resolve by hand.
- **Reports are extension-driven, not a rich query language.** `timew summary` and the bundled `totals.py` / `csv.py` / `gantt.py` extensions cover most needs; complex slicing (`hours per client per quarter, year-over-year, excluding meetings`) means writing a small Python extension that reads JSON-on-stdin from `timew export`. The contract is documented and stable, but it is your code.
- **Date / range syntax is its own DSL.** `:day`, `:week`, `:month`, `:quarter`, `:year`, `:lastweek`, `today`, `yesterday`, `monday - friday`, `2026-04-01 - 2026-04-30` all work; getting comfortable takes a session with `man timew` and the [hint table](https://timewarrior.net/docs/dates.html). Invalid ranges silently produce empty reports — always sanity-check.
- **`timew undo` only undoes the last operation.** For multi-step recovery, edit the monthly data file directly (it is plain text) and keep `~/.timewarrior/` under version control as a safety net.
- **Hooks run synchronously.** A slow `on-modify` hook (network call, large `task` query) makes every Taskwarrior write feel sluggish. Keep hook scripts fast or move them off the synchronous path.
- MIT ([COPYING](https://github.com/GothenburgBitFactory/timewarrior/blob/develop/COPYING)) — permissive; safe for commercial / billable use; no telemetry; no network calls of any kind from the binary itself. The data file is yours and lives only on the machine.

## Example invocations

```bash
# Install
brew install timewarrior
# or
apt install timewarrior

# Start a tagged interval
timew start client:acme project:billing-v3 feature-x
# ... work ...
timew stop

# Resume the most-recently-stopped interval (post-lunch one-liner)
timew continue

# Track a past interval retroactively (forgot to start)
timew track 9am - 11:30am client:acme meeting standup
timew track yesterday 14:00 - 17:00 project:billing-v3

# What's currently being tracked?
timew

# Reports
timew summary                       # today
timew summary :week                  # this week
timew summary :month client:         # this month, grouped by client:* tags
timew summary :lastweek :ids         # last week, with interval IDs visible

# Per-day breakdown
timew day
timew week

# Add or remove tags from past intervals
timew tag @3 dnd
timew untag @3 interrupted

# Export to JSON for downstream tooling
timew export :week > week.json
jq '[ .[] | select(.tags | index("client:acme")) | (.end // "now") ] | length' week.json

# Bundled extensions
timew report totals :week
timew report csv :month > april.csv
timew report gantt today

# Undo the last operation
timew undo

# Wire into Taskwarrior so `task start` / `task stop` drive timew
cp /opt/homebrew/share/doc/timewarrior/ext/on-modify.timewarrior \
   ~/.task/hooks/
chmod +x ~/.task/hooks/on-modify.timewarrior
task add project:billing-v3 +urgent fix the rounding bug
task 42 start
# ... work ... timew is automatically tracking with tags
#     project:billing-v3 +urgent and the task description
task 42 done

# Cap warnings (timew shouts when the day exceeds N hours)
timew config tagged.dnd.color "red on black"
timew config "limits.day" 8h
```
