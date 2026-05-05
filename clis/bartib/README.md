# bartib

> **A simple time tracker for the command line that stores
> everything as plaintext** — a Rust CLI that records "I started
> task X for project Y at 14:03," "I stopped at 15:47," and
> writes one human-readable line per event to a flat
> `bartib.log` file you fully own. Reports (today, week, month,
> by-project, by-task) are derived on demand from that log; no
> SQLite, no daemon, no cloud. Pinned to **v1.1.0** (commit
> `6812697f9d5dd479d0be839b516423143c63621b`,
> [LICENSE](https://github.com/nikolassv/bartib/blob/master/LICENSE),
> GPL-3.0).

Source: <https://github.com/nikolassv/bartib>

## TL;DR

`bartib` is what you reach for when `timewarrior` feels too
much, the browser-based trackers feel too cloud, and you just
want `bartib start "writing spec" -p design` and
`bartib stop` from any terminal — with the data sitting in a
plaintext log file you can grep, version in git, or sync via
Syncthing/Dropbox without any export step. Reports come out as
clean fixed-width tables suitable for stdout, markdown, or
piping into a spreadsheet.

## Install

```bash
# Cargo (any OS with a Rust toolchain)
cargo install bartib

# Homebrew (macOS / Linux)
brew install bartib

# Arch Linux (AUR)
yay -S bartib

# point bartib at a log file (one-time)
echo 'export BARTIB_FILE=$HOME/.bartib.log' >> ~/.zshrc
source ~/.zshrc
touch "$BARTIB_FILE"

# verify
bartib --version    # bartib 1.1.0
```

## License

GPL-3.0 — see [LICENSE](https://github.com/nikolassv/bartib/blob/master/LICENSE).
Copyleft. Personal time tracking is unaffected; redistribution
of modified binaries must remain GPL-3.0.

## One Concrete Example

```bash
# 1. start a task under a project
bartib start -d "writing the design spec" -p website-redesign

# 2. what am I doing right now?
bartib current

# 3. stop the active task
bartib stop

# 4. start a *different* task — auto-stops the previous one
bartib start -d "code review for PR 412" -p website-redesign

# 5. today's report (project + task + duration table)
bartib today

# 6. last week, grouped by project
bartib last week --project

# 7. month report as CSV (pipe to spreadsheet / invoice tool)
bartib report --from 2026-04-01 --to 2026-04-30 > april.txt

# 8. continue the last task you were on (after a meeting interrupt)
bartib continue

# 9. retroactive entry (forgot to start it; backdate by an hour)
bartib start -d "incident triage" -p ops --time "1h ago"

# 10. edit the log directly — it's just text
$EDITOR "$BARTIB_FILE"
```

## Niche It Fills

**Plaintext-first CLI time tracking.** The space splits three
ways: GUI/web trackers (Toggl, Clockify — fine, but you must
trust a vendor with timestamps of your day), heavy CLIs with
their own DB (`timewarrior`, `hledger-time`, `gtm` — powerful,
opinionated, schema lock-in), and "log file as truth" trackers
(`utt`, `bartib`, `watson` JSON). `bartib` is the most
ergonomic of the third group: stable plaintext format, rich
report subcommands, single static binary, no Python runtime to
manage.

## Why use it

1. **Plaintext log is the source of truth.** The file is
   diff-able, grep-able, git-versionable, Syncthing-syncable,
   and editable in `vim` when you forgot to stop a timer at
   18:00 yesterday. No proprietary export step ever.
2. **Implicit stop on next start.** Real days are
   "task A → task B → task C," not "stop A, start B, stop B,
   start C." `bartib start` ends the previous task
   automatically, which removes the most common forget-to-stop
   class of bug in time trackers.
3. **Reports are tables, not dashboards.** `bartib today`,
   `bartib last week`, `bartib report --from X --to Y` all
   render as fixed-width tables that paste cleanly into Slack,
   email, or an invoice — without launching a browser.

## Vs Already Cataloged

- **Vs [`timew`](../timew/) / [`timewarrior`](../timewarrior/):**
  timewarrior is the heavyweight in this niche — tags, hints,
  rich constraints, but a learning curve and a binary database
  format. `bartib` trades expressiveness for "your data is a
  text file you can edit." Pick `timewarrior` for complex
  billing rules and overlap detection; pick `bartib` for
  daily personal tracking with no schema you don't own.
- **Vs [`gtm`](../gtm/):** gtm is *automatic* (it watches your
  editor and infers time spent per file via git hooks).
  `bartib` is *manual* (you tell it what you're doing).
  Orthogonal — many people run both: gtm to measure, bartib to
  *categorize*.
- **Vs [`pomo`](../pomo/) / [`porsmo`](../porsmo/) /
  [`countdown`](../countdown/):** these are pomodoro / focus
  timers, not time *trackers*. They tell you when 25 minutes
  is up; `bartib` tells you what you spent the last 8 hours on.
- **Vs [`taskwarrior`](../taskwarrior/) /
  [`dstask`](../dstask/) / [`ttdl`](../ttdl/):** task managers
  (what's *to do*), not time trackers (what's *done*). Pair
  one with `bartib`: taskwarrior for the queue, bartib for the
  log.

## Caveats

- **Single active task at a time.** `bartib` models a serial
  workday. If you genuinely multitask (rare, despite what we
  all tell ourselves), you'll need to stop/start more, or
  consider `timewarrior` which supports overlapping intervals.
- **No reminders / nudges.** `bartib` will happily let you
  "track" a task for 14 hours because you forgot to `stop`
  before going home. Pair with a calendar nudge or a shell
  prompt that shows `bartib current`.
- **Manual entry — garbage in, garbage out.** Unlike `gtm`
  (editor-watching), `bartib` only knows what you tell it. If
  you forget to `start` for a meeting, it never happened.
- **Log file format is line-oriented and append-only.**
  Reordering or correcting historical entries means editing the
  file in `$EDITOR` directly — there's no `bartib edit
  --task-id N` interactive editor. This is intentional (the
  text file is the API) but trips up users expecting a CRUD
  CLI.
- **Single-machine by default.** Sync the log file across
  machines with Syncthing / git / Dropbox — `bartib` itself
  has no sync layer. Concurrent edits on two machines can
  produce log conflicts that need manual merge.
