# dstask

> **A single-binary task tracker that stores every task as one
> Markdown / YAML file in a git repo** — `dstask` is the "no
> daemon, no SQLite, no cloud account" answer to Taskwarrior:
> each task is a `<uuid>.yml` file under `~/.dstask/`, the whole
> store is a normal git repo, sync is `dstask sync` (which is
> `git pull --rebase && git push` underneath), and every mutation
> auto-commits with a message describing the change so the audit
> trail *is* `git log`. Pinned to **v1.0.1**
> ([LICENSE](https://github.com/naggie/dstask/blob/master/LICENSE),
> MIT).

Source: <https://github.com/naggie/dstask>

## TL;DR

`dstask add fix login bug project:web +bug priority:P1` creates a
file, auto-commits, ready. `dstask` (no args) prints the
priority-sorted next-actions list. Multi-device sync is `dstask
sync` against any git remote (GitHub, Gitea, self-hosted) — no
proprietary server, no per-seat SaaS.

## Install

```bash
# Homebrew (macOS / Linux)
brew install dstask

# Go
go install github.com/naggie/dstask/cmd/dstask@v1.0.1

# Pre-built release binary
curl -LO https://github.com/naggie/dstask/releases/download/v1.0.1/dstask-darwin-arm64
sudo install dstask-darwin-arm64 /usr/local/bin/dstask

# bash / zsh / fish completions
dstask _completions zsh > ~/.zsh/completions/_dstask

# verify
dstask version    # 1.0.1
```

First run creates `~/.dstask/` and initialises a git repo there;
point `git remote add origin <url>` at any git host and `dstask
sync` from then on.

## License

MIT — see
[LICENSE](https://github.com/naggie/dstask/blob/master/LICENSE).
Permissive, redistribute and modify freely.

## One Concrete Example

```bash
# 1. add a task with project, tag, priority
dstask add write release notes project:rel-2.5 +docs priority:P2

# 2. show next actions (priority + age sorted)
dstask
# ID  Pri  Project  Summary                                   Tags
#  1  P1   web      fix login bug                             bug
#  2  P2   rel-2.5  write release notes                       docs

# 3. start a task (logs a started: timestamp)
dstask start 1
# ... work ...
dstask stop 1
dstask done 1

# 4. note-attach a multi-line markdown body
dstask 2 note  # opens $EDITOR on the task's note section

# 5. context filter (only show this project until cleared)
dstask context project:web
dstask                 # filtered list
dstask context none    # clear

# 6. multi-device sync
dstask sync            # git pull --rebase && git push
```

## Niche It Fills

**Single-binary, file-per-task, git-as-sync personal task
manager.** Taskwarrior's data model and ergonomics, but without
the binary `.data` files that fight with `git diff`, without the
optional `taskd` server, and without the C++ + per-distro
package nightmare — one Go binary on every host, one git remote
shared between them.

## Why use it

1. **Every task is human-readable on disk.** `cat
   ~/.dstask/pending/abc-123.yml` shows a normal YAML doc with
   the body as Markdown; you can `grep`, `sed`, `rsync`, or pull
   up a task from any machine without `dstask` installed.
2. **Sync is just git.** No proprietary protocol. Use GitHub for
   personal, a self-hosted Gitea / Forgejo for the team
   shared-task variant, or a USB stick for the air-gapped
   variant. Conflicts resolve with normal `git mergetool`.
3. **Auto-commit is the audit log.** Each `add` / `modify` /
   `done` writes a commit; `git log --oneline ~/.dstask/` is
   "everything I did to my tasks this week" with no extra
   tooling.

## Vs Already Cataloged

- **Vs [`taskwarrior`](../taskwarrior/):** taskwarrior is the
  feature-complete original — recurrence, urgency formula,
  hooks, `taskd` sync server, decades of plugins. dstask is the
  smaller, simpler, git-native subset: one binary, no daemon,
  files-as-storage, ~80 % of the daily-driver feature set.
  Pick taskwarrior for advanced urgency / recurrence math; pick
  dstask if you already live in git and want the data to look
  like every other repo on disk.
- **Vs [`taskbook`](../taskbook/) / [`dooit`](../dooit/) /
  [`ttdl`](../ttdl/):** these are pretty TUIs / boards over
  local JSON or todo.txt, single-machine by default. dstask's
  win is the multi-device git-sync story plus the audit-log
  guarantee.
- **Vs [`gtm`](../gtm/) (git time-tracker):** gtm logs *time
  spent* against commits; dstask logs *tasks* whose sync
  *happens* to be git. Complementary — track tasks in dstask,
  measure time on the resulting commits with gtm.

## Caveats

- **Single-user-per-repo assumption.** Multiple humans pushing
  to the same `~/.dstask/` repo work but auto-commit messages
  cross-pollinate and conflict resolution requires `git`
  literacy. For team shared tasks the maintainer recommends
  one repo per person plus a shared "ideas" repo, not one repo
  for the whole team.
- **No recurrence engine.** Recurring tasks are emulated by
  hand or by a cron job that re-creates them; if "every Tuesday
  pay rent" is a hard requirement, taskwarrior's `recur:`
  field is more ergonomic.
- **No web UI / mobile app.** The `~/.dstask/` checkout has to
  live on the device you want to read from. Some users keep a
  worktree on a phone via Termux + git; there is no
  first-party iOS / Android client.
- **YAML schema is stable but not versioned for migrations.**
  Roll-your-own scripts that mutate the files directly will
  survive across versions but are not covered by the
  compatibility promise — go through the CLI for anything
  scripted.
