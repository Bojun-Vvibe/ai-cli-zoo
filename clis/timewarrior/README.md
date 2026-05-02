# timewarrior

> **Command-line time tracking** — start / stop / tag intervals,
> annotate them after the fact, query and chart the result.
> Sibling project to `taskwarrior`, designed to live in the
> shell instead of a calendar app.
> Pinned to **v1.9.1**
> ([LICENSE](https://github.com/GothenburgBitFactory/timewarrior/blob/develop/LICENSE),
> MIT).

Source: <https://github.com/GothenburgBitFactory/timewarrior>

## TL;DR

`timew` records *intervals*: a start timestamp, an optional end
timestamp, and a set of free-form tags. You bracket your work
with `timew start <tags>` and `timew stop`, or backfill an
interval after the fact with `timew track <range> <tags>`. The
data lives as plain-text files under `~/.timewarrior/`, queryable
with built-in reports (`day`, `week`, `summary`, `gaps`,
`month`) and extensible via shell-script "extensions" that
receive the interval set on stdin and emit a report on stdout.
No daemon, no DB, no web UI.

## Install

```bash
# Homebrew (macOS / Linuxbrew)
brew install timewarrior

# Debian / Ubuntu
sudo apt install timewarrior

# Arch
sudo pacman -S timew

# Fedora
sudo dnf install timew

# from source (CMake)
git clone --recursive \
    https://github.com/GothenburgBitFactory/timewarrior && cd timewarrior
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j && sudo cmake --install build

# verify
timew --version    # 1.9.1
```

## License

MIT — see
[LICENSE](https://github.com/GothenburgBitFactory/timewarrior/blob/develop/LICENSE).
Permissive; embed in internal tooling without copyleft concern.

## One Concrete Example

```bash
# 1. start tracking — current interval gets these tags
timew start project-x review pr-1234

# 2. context switch — stops the previous interval, starts a new one
timew start project-y meeting

# 3. stop tracking entirely
timew stop

# 4. backfill an interval you forgot to track
timew track 09:00 - 10:30 project-x deep-work

# 5. tag retroactively (last interval)
timew tag @1 billable client-acme

# 6. reports
timew day                    # today, hour-by-hour grid
timew week project-x         # this week, filtered to one tag
timew summary :month         # tabular totals by tag for the month
timew month project-x :ids   # month chart filtered + with IDs

# 7. ad-hoc query for invoicing
timew summary 2026-04-01 - 2026-04-30 client-acme billable
# emits a table per-day-per-tag with totals; pipe to `awk` /
# `column -t` / a custom extension for invoice CSV.

# 8. extensions: drop a script into ~/.timewarrior/extensions/
#    that reads the interval JSON on stdin and emits whatever
#    report shape you want. Bundled extensions: totals.py,
#    csv.py, gantt.py.
timew report csv :month > april.csv
```

## Niche It Fills

**The "time tracking that lives where the work happens" gap.**
The dominant alternatives are calendar apps (Toggl, Harvest,
Clockify) — web UIs you context-switch into, with their own
auth, sync, and "did the timer actually start" failure modes.
For developers who already live in the terminal, `timew` is a
local, file-backed, scriptable timer: starting an interval is
one command in the shell you're already in, and the data is
plain text you can grep, version, back up, and analyze with the
same tools you use for everything else. It is intentionally not
a project-management or task system — `taskwarrior` is the
sibling for that, and they integrate via a hook.

## Why use it

Three concrete things `timew` does that the SaaS timers don't:

1. **Local plain-text store.** `~/.timewarrior/data/` is a tree
   of monthly text files; you can `git` it for history, `grep`
   it for "what did I work on last March", or rewrite it by
   hand if you mistyped a tag. No vendor lock-in, no export
   format, no "the API rate-limited me".
2. **Scriptable reports.** Extensions are stdin/stdout filters,
   so a custom report (e.g. "billable hours per client per
   week as CSV for invoice generation") is a 30-line shell or
   Python script, not a paid Pro-tier feature.
3. **`taskwarrior` integration.** With the `on-modify.timewarrior`
   hook, starting / stopping a `task` automatically starts /
   stops a matching `timew` interval with the task's tags. So
   "what did I work on" comes for free from "what was I doing
   in `task`", with zero additional muscle memory.

For an LLM-CLI workflow, `timew` is the **session-accounting
substrate**: an agent can read `timew export :week --json` to
understand how the human spent the week, summarize it for a
status update, draft an invoice, or surface "you have logged
0 hours on project-x but 3 PRs were merged — should we
backfill?". All from local plain-text data, no OAuth dance.

## Vs Already Cataloged

- **Vs [`taskwarrior`](../taskwarrior/) (sibling project):**
  Different axes: `task` tracks *what to do* (todo items with
  due dates, priorities, deps); `timew` tracks *what was
  done* (intervals with tags). They share a maintainer and an
  integration hook. Most users run both: `task` for the queue,
  `timew` for the clock.
- **Vs [`watson`](../watson/):** Watson is the closest direct
  competitor — Python, MIT, CLI time tracker, similar mental
  model (`watson start project +tag`). Differences: Watson
  has built-in optional sync to a remote backend (Crick); `timew`
  is local-only by design. Watson stores in a single JSON file;
  `timew` stores in monthly text files (better for very long
  histories). Watson is in maintenance mode; `timew` had its
  most recent release in 2025-08. Pick `timew` if you want
  active maintenance + extension hooks; pick `watson` if the
  smaller surface fits your taste.
- **Vs [`hledger`](../hledger/) `time` (timeclock format):**
  hledger reads a `timeclock` file (`i 2026-04-22 09:00 acme`
  / `o 2026-04-22 10:30`) and reports on it like a ledger. It's
  great if you already use hledger for money and want one tool;
  `timew` is better if you want first-class start/stop,
  retroactive tagging, gap detection, and a chart-style `day`
  / `week` view rather than ledger-style register output.
- **Vs a `cron` + log-file homegrown solution:** "Just `date >>
  ~/log` when I start" gets you nothing for free: no overlap
  detection, no per-tag aggregation, no gap reports. `timew`
  is what that idea grows into after the third evening of
  reinventing it.
- **Vs Toggl / Harvest / Clockify (SaaS):** Those win on
  mobile, on team billing dashboards, and on auto-detection
  via desktop agents. `timew` wins on local-first, scriptability,
  and "no monthly fee for a feature I want behind the Pro
  paywall".

## Caveats

- **No mobile / web UI.** If you start a session at the laptop
  and switch to phone, the timer doesn't follow you. For
  mobile-first tracking, the SaaS options are unavoidable.
- **No auto-detection of activity.** It will happily record an
  8-hour interval that you actually spent in a long meeting
  unrelated to the tag. Build the habit of `timew stop`, or
  add a screen-locker hook to stop on lock and resume on
  unlock.
- **Tag space is flat and free-form.** Typo `bilable` instead
  of `billable` and your invoicing report misses an hour;
  there is no controlled vocabulary or autocomplete out of
  the box. Define the tag set in a comment in your invoice
  extension or wrap `timew start` in a shell function that
  validates.
- **Reports are text/Unicode bar charts.** Beautiful in a
  monospace font, harder to share in a doc. Use `timew export
  :week --json` and feed it into `gnuplot` / `vega-lite` /
  a notebook for graphics.
- **No team / multi-user model.** This is single-user
  personal time tracking. For team time visibility, sync
  individual `timew` exports into a shared spreadsheet, or
  pick a SaaS.
