# mprocs

> **Run multiple long-lived shell commands in parallel under one TUI
> with a process list on the left and the focused process's
> stdout / stderr on the right — `tmux` for `npm run dev` + `cargo
> watch` + `docker compose up` + log tail without learning `tmux`.**
> Pinned to **v0.9.2**, MIT
> ([LICENSE](https://github.com/pvolok/mprocs/blob/master/LICENSE)).

- **Repo:** https://github.com/pvolok/mprocs
- **Latest version:** v0.9.2
- **License:** MIT (`LICENSE` at repo root, SPDX `MIT`)
- **Category:** `dev-experience` / `process-runner`
- **Language:** Rust

## What it does

`mprocs` reads a `mprocs.yaml` (or accepts ad-hoc commands on argv)
declaring a list of long-running processes — each with `cmd`, `cwd`,
`env`, optional `autostart` / `autorestart` / `stop` (signal to send
on quit) — and boots a two-pane TUI: process list on the left
(name + status colour: running / stopped / exited-ok / exited-fail),
the focused process's combined stdout+stderr scrollback on the right
with ANSI colour preserved. Keystrokes: `↑`/`↓` switch focus, `s`
start, `x` stop (sends configured signal then `SIGKILL` after
timeout), `r` restart, `c` copy mode (vim-shaped scroll + select),
`a` send keys to the underlying PTY (so `npm run dev`'s `r` to
restart works), `q` quit (stops every process gracefully). Per-process
`stop` config supports `SIGINT` / `SIGTERM` / `SIGKILL` / a literal
keystroke string (`q\n`) for programs that own a TTY and need a
specific quit input. `mprocs --names api,web,db` plus three trailing
`--` blocks runs ad-hoc without a YAML.

## Why included

The dev-loop spawns five terminals: `pnpm dev` for the frontend,
`cargo run` for the backend, `docker compose up postgres redis`,
a `wrangler tail` log stream, and a `pytest --watch` for the tests.
`tmux` solves it but the per-pane setup is muscle memory; a Procfile
under `foreman` / `overmind` is closer but the output is interleaved
into one stream that's unreadable when two services are noisy at
once. `mprocs` is the dedicated answer: one YAML, one process per
named row, isolated scrollbacks switchable in one keystroke,
restart-on-crash optional per row, graceful shutdown on `q` so the
local Postgres container actually stops instead of leaking. Pairs
with [`zellij`](../zellij/) (run `mprocs` in one zellij pane
alongside the editor and a shell — different layer: zellij multiplexes
*terminals*, mprocs multiplexes *processes*) and orthogonal to
[`pueue`](../pueue/) (queue of one-shot jobs vs. dashboard of
long-lived dev processes).
