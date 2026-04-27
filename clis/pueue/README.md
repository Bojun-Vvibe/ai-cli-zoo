# pueue

> **A command-line task scheduler for long-running shell
> commands** — a Rust daemon (`pueued`) plus client (`pueue`)
> that queues, parallelises, pauses, and resumes shell jobs so
> you can fire-and-forget without nohup / tmux / screen
> juggling. Pinned to **v4.0.4**
> ([LICENSE](https://github.com/Nukesor/pueue/blob/main/LICENSE),
> MIT).

Source: <https://github.com/Nukesor/pueue>

## TL;DR

`pueue` is what you reach for when you'd otherwise prefix five
shell commands with `nohup ... &` and lose track of which one
finished, which one failed, and what the output was. Submit a
job with `pueue add -- ./train.sh` and it runs in a managed
queue with configurable parallelism per group; `pueue status`
shows everything, `pueue log <id>` shows captured stdout/stderr,
`pueue follow <id>` tails it live, and the daemon survives your
SSH session ending. Groups let you pin "at most 2 GPU jobs at a
time, but unlimited CPU jobs." `pueue pause` stops the queue,
`pueue start` resumes, `pueue restart --in-place <id>` re-runs
a failure with the same env.

## Install

```bash
# Homebrew (macOS / Linux)
brew install pueue

# Linux package managers
# Arch:   pacman -S pueue
# Nix:    nix-env -iA nixpkgs.pueue

# Cargo (any OS with a Rust toolchain)
cargo install --locked pueue

# verify
pueue --version     # pueue 4.0.4
pueued --version    # pueued 4.0.4

# start the daemon (runs in background; or use a systemd / launchd unit)
pueued -d

# submit a job
pueue add -- "sleep 10 && echo done"
pueue status
```

## License

MIT — see
[LICENSE](https://github.com/Nukesor/pueue/blob/main/LICENSE).
Permissive, no attribution required for binaries.

## Niche It Fills

**`nohup &` + a notebook, but persistent and queryable.** The
shell gives you `&` for background and `nohup` for surviving
logout, but loses the output, the exit code, and any sense of
"how many of these are running right now." `pueue` is the
missing job queue: persistent across reboots, with captured
output, exit codes, groups, parallelism limits, and a real
status command.

## Why it pairs with coding agents

For agent workflows that kick off long jobs (training, eval,
batch summarisation, repo-wide refactors, full-test runs),
`pueue` gives the agent a place to put those jobs and come
back to them later without holding a shell open. The agent can
`pueue add` a 30-minute eval, return immediately, do other work,
then `pueue log <id>` when the result is needed. Group-level
parallelism caps prevent an agent from accidentally launching
20 concurrent eval runs on a 4-GPU box.

## Vs Already Cataloged

- **Vs `tmux` / `screen`:** terminal multiplexers give you
  *windows* for long jobs, but no queue, no exit-code tracking,
  no parallelism cap. `pueue` complements them — keep `tmux` for
  interactive shells, use `pueue` for batch jobs that should
  outlive the session.
- **Vs `make -j` / `xargs -P`:** those parallelise *one* batch
  invocation. `pueue` is a long-lived queue you keep adding to
  over hours and days.

## Caveats

- **Daemon must be running.** `pueued` is a separate process; if
  it crashes or you forget to start it, `pueue add` errors out.
  Use a `systemd --user` unit on Linux or a `launchd` plist on
  macOS to keep it up.
- **Captured output is on disk, not a database.** Logs are
  per-task files under `~/.local/share/pueue/`; long-running
  high-volume stdout can fill a disk if you forget to clean.
- **Not a distributed scheduler.** Single-host. For
  multi-machine job orchestration look at Nomad / k8s Jobs /
  Slurm — `pueue` is deliberately scoped to "one box, many
  shells."
