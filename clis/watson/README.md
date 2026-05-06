# watson

> **watson** — TailorDev's command-line time tracker: start / stop a
> timer tagged with a project + arbitrary tags, store frames in a flat
> local JSON file, and produce reports / aggregates from the CLI.
> Pinned to **2.1.0**, MIT — license file:
> [LICENSE](https://github.com/TailorDev/Watson/blob/master/LICENSE).

Source: <https://github.com/TailorDev/Watson>

## TL;DR

`watson start client-x +meeting +planning` opens a timer tagged with
a project (`client-x`) and any number of `+tags`. `watson stop`
closes it and appends a *frame* (start, stop, project, tags) to
`~/.config/watson/frames`. Everything else — `watson status`,
`watson log`, `watson report`, `watson aggregate`, `watson projects`,
`watson tags`, `watson edit` — is a query / mutation over that flat
file, with filters by date range, project, tag, day / week / month
buckets, and JSON output for scripting.

The model that makes watson useful for engineers (vs. dozens of
SaaS time trackers) is "the source of truth is a plain JSON file in
`$XDG_CONFIG_HOME/watson/`": you can `git`-track it, `jq` over it,
sync it across machines via Syncthing / iCloud / Dropbox, and edit
it by hand when you forgot to stop the timer at the end of the day.
There is an optional sync server (`watson sync`) that ships frames
to a Crick-compatible HTTP API for multi-machine aggregation, but
the local-first JSON-on-disk design is the default and the safe one.

## Install

```bash
# pip / pipx (the canonical install path)
pipx install td-watson

# Homebrew
brew install watson

# Pre-built distribution from PyPI / GitHub
# https://github.com/TailorDev/Watson/releases/tag/2.1.0
```

`watson` is a Python application with a small dependency surface
(click, requests, arrow). The `pipx` install gives you an isolated
environment — recommended over a global `pip install`.

## Example commands

```bash
# Start a timer
watson start client-x +meeting +planning

# Show the active timer
watson status

# Stop and persist the frame
watson stop

# What did I do this week?
watson log --week

# Time per project for the past 7 days, table form
watson report --from "$(date -v-7d +%Y-%m-%d)"

# Same, machine-readable
watson report --json --from 2026-04-01 --to 2026-04-30

# Aggregate by tag for last month
watson aggregate --month --tag meeting

# Edit the last frame (corrects a forgotten-to-stop timer)
watson edit

# Add a frame after the fact (yesterday afternoon's meeting)
watson add --from "2026-05-05 14:00" --to "2026-05-05 15:30" \
  client-x +meeting

# List all known projects / tags
watson projects
watson tags
```

Bash / zsh / fish completion scripts ship in the package
(`watson --shell-completion install <shell>`) so `watson start <Tab>`
completes against the project list extracted from the frames file.

## Niche it occupies

**Plain-text command-line time tracker** with a flat-JSON local
store — closest catalogue neighbours are [`bartib`](../bartib/),
[`timew`](../timew/) / [`timewarrior`](../timewarrior/), [`gtm`](../gtm/),
and [`dstask`](../dstask/) (the latter two are different shapes of
the time-tracking / task-tracking overlap).

- Pick **watson** when you want a *single project + free-form tags*
  per frame, a Python install path (so it lives next to `pipx`-managed
  CLIs), and a JSON-on-disk store you can sync via your own dotfile /
  cloud-folder mechanism. The `aggregate` and `report` flag surface
  is the cleanest of the lot for "how much did I bill client X this
  month", and the optional Crick sync server is there if you grow
  out of single-machine.
- Pick [`timewarrior`](../timewarrior/) (`timew`) when you want a
  C++ binary with no Python runtime, a richer interval-algebra
  query language (`timew summary :ids :annotations from monday to
  friday`), and tight integration with [`taskwarrior`](../taskwarrior/)
  (`task start <id>` auto-starts a `timew` interval). timew is the
  more powerful query engine; watson is the simpler ergonomic.
- Pick [`bartib`](../bartib/) when you want a single static Rust
  binary, a flat human-readable text log (not JSON), and the lowest
  install + sync friction across machines. Bartib is "watson but in
  Rust with a text log instead of JSON".
- Pick [`gtm`](../gtm/) when the verb is *git-aware* time tracking
  — gtm hooks into git to attribute time to commits. watson tracks
  arbitrary work; gtm tracks specifically *coding* work tied to
  git activity. They compose: gtm for billable code, watson for
  the meetings / writing / planning around it.
- Orthogonal to [`taskwarrior`](../taskwarrior/) / [`dstask`](../dstask/)
  / [`ttdl`](../ttdl/) (those are *task* trackers — what to do —
  whereas watson is a *time* tracker — how long it took). Run both:
  task tracker for the queue, watson for the bill.

Pairs cleanly with [`hledger`](../hledger/) / [`beancount`](../beancount/)
(plain-text accounting CLIs that ingest CSV / JSON — `watson report
--json` feeds straight into a `journal` for time-as-money
bookkeeping) and with [`chezmoi`](../chezmoi/) / [`yadm`](../yadm/)
(version-control the frames file across machines without running
the sync server).

## Caveats

- Last release **2.1.0** (March 2022). The project is in a
  feature-stable / bug-fix-only state — pin the version and treat
  it as mature, not actively-developed.
- The Crick sync server (`watson sync`) is community-maintained and
  not officially hosted; the local-first JSON-on-disk path is the
  recommended one.
- The frames file is plain JSON, so concurrent edits across
  machines without a sync layer cause merge conflicts — pick one
  of {single-machine, Crick server, dotfile-repo + manual merge,
  Syncthing/Dropbox single-writer-at-a-time}.
- Time zones are stored as offsets at frame time; moving across
  zones mid-frame yields the right wall-clock times but be aware
  when re-aggregating across DST boundaries.

## Citation

- Repo: <https://github.com/TailorDev/Watson>
- Latest release: **2.1.0**
- License: **MIT**
- License file: [LICENSE](https://github.com/TailorDev/Watson/blob/master/LICENSE)
