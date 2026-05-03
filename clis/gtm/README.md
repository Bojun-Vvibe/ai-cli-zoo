# gtm

> **Automatic, git-native time tracking for the actual files you
> edit** — a Go binary that hooks into your editor and into git
> to record the wall-clock time you spend on each file, then
> stores those minutes in git notes attached to the commit that
> contained the work, so the time data travels with the repo and
> survives clones, rebases, and history rewrites the same way the
> code does. Pinned to **v1.3.5**
> ([LICENSE](https://github.com/git-time-metric/gtm/blob/master/LICENSE),
> MIT).

Source: <https://github.com/git-time-metric/gtm>

## TL;DR

`gtm` (Git Time Metrics) is an opt-in time tracker that listens
for editor "buffer focused" events from a small set of supported
editors (Vim, Emacs, VS Code, Sublime, Atom, IntelliJ, Xcode via
plugins), accumulates per-file edit time in 60-second windows,
and at `git commit` time writes the totals as a structured note
on the commit (`refs/notes/gtm-data`). Reports (`gtm report`,
`gtm status`) read those notes back. Because the data is in git
notes, two collaborators on the same repo can each see their own
time *and* the team's combined time without a server, a SaaS
account, or a manual start/stop timer.

## Install

```bash
# Homebrew (macOS / Linux)
brew tap git-time-metric/gtm
brew install gtm

# Pre-built binary (Linux / macOS / Windows)
# Pick from https://github.com/git-time-metric/gtm/releases/tag/v1.3.5
curl -L -o gtm.tar.gz \
  https://github.com/git-time-metric/gtm/releases/download/v1.3.5/gtm.v1.3.5.darwin.amd64.tar.gz
tar xf gtm.tar.gz
sudo install gtm /usr/local/bin/

# verify
gtm verify ">= 1.3.5"

# 1) Initialize gtm in each repo where you want tracking:
cd ~/code/myrepo
gtm init           # installs git hooks (.git/hooks/post-commit etc.)

# 2) Install the plugin for your editor (one-time, per editor):
#   Vim:        https://github.com/git-time-metric/gtm-vim-plugin
#   VS Code:    install the "GTM Plus" or "GTM" extension
#   Emacs:      https://github.com/git-time-metric/gtm-emacs-plugin
#   Sublime:    https://github.com/git-time-metric/gtm-sublime3-plugin
#   IntelliJ:   https://github.com/git-time-metric/gtm-jetbrains-plugin
#   Atom:       https://github.com/git-time-metric/gtm-atom-plugin
```

Once both halves are installed, `gtm` records time silently while
you edit. Nothing to start, nothing to stop.

## License

MIT — see
[LICENSE](https://github.com/git-time-metric/gtm/blob/master/LICENSE).
Permissive; the binary is freely redistributable, the editor
plugins are individually MIT/Apache from the same org.

## One Concrete Example

```bash
# Day-of inspection — what have I been working on right now?
gtm status
# > today  1h45m  feat/payment-rewrite
#     45m   payments/charge.go
#     30m   payments/charge_test.go
#     20m   docs/payments.md
#     10m   .gtm/                    (overhead)

# After committing, see the time attached to the commit
git commit -am "wire stripe charge"
gtm report -last-commit
# > 1h45m  3a8f2c1  wire stripe charge
#     45m  payments/charge.go
#     30m  payments/charge_test.go
#     20m  docs/payments.md

# Weekly summary across the whole repo
gtm report -from yesterday
gtm report -from "2026-04-25" -to "2026-05-02"

# Time per author (after teammates push their notes)
gtm report -authors                 # group by commit author
gtm report -authors -tags refactor  # filter by commit message tag

# Per-file totals across history
gtm report -files

# Make sure your notes get pushed and pulled with normal git ops
# (one-time, per clone):
git config --add remote.origin.fetch '+refs/notes/gtm-data:refs/notes/gtm-data'
git config --add remote.origin.push  '+refs/notes/gtm-data:refs/notes/gtm-data'

# Now `git push` / `git pull` includes time notes alongside commits
git push
git pull
```

## Niche It Fills

**Time tracking that lives in the repository, not in a SaaS
dashboard.** Most time trackers (Toggl, Harvest, RescueTime,
Clockify) are wall-clock-of-the-human ("I worked 8h today")
stored on a vendor's server, attributed to a project tag the
human had to pick. `gtm` is wall-clock-of-the-file ("this commit
took 1h45m, broken down 45m / 30m / 20m / 10m by file") stored
as git notes on the commit, attributed automatically by which
editor buffer was focused. That makes it the right tool for
estimating how long a specific change actually took, for
reviewing pull requests with effort context, or for billing on
a per-feature basis where "feature" means "set of commits".

## Why use it

Three things `gtm` does that timer-based trackers do not:

1. **Zero ongoing user action after install.** No start/stop
   button, no project tag to pick, no Pomodoro alarm to
   acknowledge. The editor plugin emits a focus event, the daemon
   accumulates a 60-second window, and the post-commit hook
   writes the note. The cost of capturing data is zero per
   commit, so the data is actually captured.
2. **Per-file granularity, not per-session.** A 4-hour session
   that touched 12 files becomes 12 numbers, not one. That
   matches how code review and refactoring questions are
   actually framed ("how much of this PR's time went into the
   test file vs the migration?") instead of forcing the human to
   re-bin a flat 4-hour block.
3. **Data lives in git, not in a vendor.** Time totals push and
   pull alongside the code. Cloning the repo on a new laptop
   brings the history back; deleting the repo deletes the time
   data; there is no account to cancel and no exporter to write
   when you leave the team.

For an LLM-CLI workflow that summarizes "what did this PR
change", `gtm report -last-commit` adds the missing effort axis
to the diff axis ("the migration is +400 lines but only 12
minutes; the new validator is +60 lines and 90 minutes — review
the validator carefully").

## Vs Already Cataloged

- **Vs [`timewarrior`](../timewarrior/):** orthogonal —
  `timewarrior` is a manual start/stop interval tracker
  (`timew start coding`, `timew stop`) with rich tagging, range
  queries, and reports, stored in `~/.timewarrior/`. It tracks
  the *human*, not the *files*. Use `timewarrior` when you bill
  by interval and the project boundaries are mental
  ("admin time", "deep work"); use `gtm` when you want the data
  attached to specific commits and files.
- **Vs [`gtm`'s peer `wakatime` (not cataloged):** `wakatime`
  has the same editor-plugin → time-per-file model but stores
  the data on `wakatime.com` (free tier covers individuals,
  team features paid). `gtm` is the offline, no-account, lives-
  in-the-repo alternative; pick `wakatime` if you want the
  hosted dashboard and language/project leaderboards, pick `gtm`
  if you want the data inside the repo and on no third-party
  server.
- **Vs [`taskwarrior`](../taskwarrior/) /
  [`taskwarrior-tui`](../taskwarrior-tui/):** orthogonal —
  taskwarrior tracks *tasks* (todo items with priority, due
  date, dependencies), not time spent on files. Pair them: use
  taskwarrior to plan, use `gtm` to measure.
- **Vs `git log --stat` / `git shortlog`:** native git
  attribution is line-count and authorship, not time. `gtm`
  fills in the missing time column without changing the rest of
  the git workflow.

## Caveats

- **Editor plugin coverage is the floor.** Officially: Vim,
  Emacs, Atom, Sublime, VS Code, IntelliJ family, Xcode (via
  community plugin). If you spend significant time in an
  unsupported editor (modern Zed, Helix, Lapce, JupyterLab)
  that time is silently invisible.
- **One-minute granularity.** The daemon bins focus events into
  60-second windows. Quick context switches under a minute show
  up wholly under whichever file you ended the minute on; the
  totals are still close, but don't expect second-precision per
  file.
- **Git notes are easy to drop accidentally.** If contributors
  don't configure the `refs/notes/gtm-data` fetch / push specs,
  notes never leave the laptop. `gtm init` doesn't fix that on
  *other* clones — each developer must opt in. CI mirrors that
  don't fetch notes will appear empty in `gtm report`.
- **Project pace.** v1.3.5 ships current binaries (2024) and is
  stable, but commit cadence is slow — treat it as a working
  classic, not an actively-evolving product. Pick `wakatime` if
  you need a vendor with a roadmap.
- **No project / client tagging.** Every minute is attached to a
  file in a repo; there is no built-in concept of "client
  Acme" vs "client Beta". For multi-client billing, layer
  taskwarrior or timewarrior tags on top, or split work into
  separate repos.
- **Doesn't track non-editor work.** Reading docs in a browser,
  whiteboarding, pairing on someone else's machine — none of it
  registers. `gtm` measures keys-on-files, which under-counts
  designers, reviewers, and PMs.
