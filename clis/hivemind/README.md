# hivemind

## Overview

`hivemind` is a `Procfile`-based process manager for development that
runs a small set of long-lived processes (web server, worker, scheduler,
asset watcher, log tail, REPL) as one foreground tree, multiplexing
their stdout / stderr into a single colour-prefixed stream so you can
see at a glance which output belongs to which service. It is the
slim, dependency-free Go alternative to Ruby's `foreman`: one static
binary, no `bundle exec`, no per-language runtime, runs on macOS /
Linux / FreeBSD / Windows from the same release artefact.

A typical `Procfile` looks like:

```
web:    bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq
js:     pnpm run dev
css:    pnpm run css:watch
db:     redis-server
```

`hivemind` boots all five, prefixes each line of their output with the
process name in a stable colour (`web    | Listening on tcp://0.0.0.0:3000`),
forwards `Ctrl+C` to every child as `SIGINT` for a clean shutdown, and
exits when the first process dies (so a crashed worker takes the whole
dev session down instead of leaving zombies — the behaviour you want
for "is the dev environment healthy yes/no").

## Repo URL

https://github.com/DarthSim/hivemind

## Version

v1.1.0 (released 2021-12-03)

## License

MIT — upstream LICENSE file:
[`LICENSE`](https://github.com/DarthSim/hivemind/blob/master/LICENSE).

## Install

Homebrew (macOS / Linux):

```bash
brew install hivemind
```

Go install (any platform with Go ≥ 1.16):

```bash
go install github.com/DarthSim/hivemind@latest
```

Pre-built binary from a release tarball:

```bash
HIVEMIND_VERSION=v1.1.0
curl -Lo hivemind.gz "https://github.com/DarthSim/hivemind/releases/download/${HIVEMIND_VERSION}/hivemind-${HIVEMIND_VERSION}-$(uname | tr A-Z a-z)-amd64.gz"
gunzip hivemind.gz && chmod +x hivemind
sudo install hivemind /usr/local/bin/
```

Verify:

```bash
hivemind --version    # Hivemind version 1.1.0
```

Configuration is `Procfile` in the project root plus optional
`.hivemind.yml` for per-process env files, port-base offsets, and
sub-process command overrides:

```yaml
# .hivemind.yml
processes:
  web:
    env_files:
      - .env
      - .env.development
print-timestamps: true
port-base: 5000
```

Run with `hivemind` (defaults to `./Procfile`); or `hivemind -f
Procfile.dev` to pick a different file; or `cat Procfile.test |
hivemind -f -` to feed it from stdin (useful for ephemeral CI shapes).

## Why use it

Three things `hivemind` does that beat the bash equivalent
(`./web & ./worker & wait`) for the dev-environment-as-a-set-of-services
shape:

1. **Coalesced colour-prefixed output.** Every line is tagged with the
   producing process name in a stable per-process colour, so a stack
   trace that scrolls past three other services' logs is still
   identifiable as "from the worker" without `tail -f log/worker.log`
   in a separate pane. The colour assignment is deterministic across
   runs, so muscle memory holds across restarts.
2. **`Ctrl+C` does the right thing.** A single `SIGINT` propagates to
   every child as `SIGINT` (not `SIGKILL`), respects each process's
   shutdown handler, and `hivemind` waits for them all before exiting.
   The `bash & wait` equivalent leaks children when the parent shell
   dies; `hivemind` uses a process group so even `kill -9` on the
   parent reaps the tree.
3. **Procfile is a tiny, language-agnostic contract.** Same file works
   in `foreman` (Ruby), `honcho` (Python), `goreman` (Go), Heroku,
   Render, Railway, fly.io, and most PaaS deployment surfaces — so the
   "how does this app start in dev" answer and the "how does this app
   start in prod" answer are the same five lines, diffable in git, and
   onboarding a new contributor stops requiring a tribal-knowledge
   `make dev` recipe.

## Vs Already Cataloged

- **Vs [`overmind`](../overmind/):** same author, same `Procfile`
  contract, complementary feature set. `overmind` adds `tmux`-backed
  process isolation (`overmind connect web` drops you into the web
  process's `tmux` pane to send input — needed for Rails consoles
  reading from stdin, debugger breakpoints, etc.) at the cost of
  requiring `tmux` on the host. `hivemind` is the no-`tmux`,
  single-foreground-stream sibling — pick it when the dev workflow
  doesn't need to interact with any individual process's stdin.
- **Vs [`mprocs`](../mprocs/):** orthogonal layout. `mprocs` is a
  full-screen TUI with one pane per process and per-pane scrollback;
  `hivemind` is a flat scrolling log. `mprocs` is better for "I want
  to read the full output of one specific process while others run";
  `hivemind` is better for "I want to see the merged timeline of
  events across all services as they happen".
- **Vs [`watchexec`](../watchexec/):** orthogonal — `watchexec` re-runs
  a single command on file change, `hivemind` runs many commands in
  parallel. They compose: a `Procfile` line like `tests: watchexec -e
  py -- pytest -x` puts a file-watching test runner inside a
  `hivemind`-managed dev session.

## Caveats

- **Last release was 2021-12-03 (v1.1.0).** The project is feature-
  complete and stable rather than abandoned — the upstream still
  responds to issues and the binary works on current macOS arm64 and
  Linux amd64 — but expect no new features. If the gap is "I need
  per-process restart on file change", the answer is to wrap the
  command in `watchexec` inside the Procfile, not to wait for
  upstream.
- **Crash propagation is intentional.** When any single process exits
  (clean or crashed), `hivemind` shuts down the rest. For the dev
  workflow this is the right default (a worker that died silently is
  the bug you want to see), but for "keep the rest running while I
  debug one" you want `overmind` (which restarts on demand) or a
  process supervisor (`s6`, `runit`, systemd) instead.
- **No web UI, no metrics, no log persistence.** Output is to stdout;
  if you want history, pipe `hivemind` into `tee` or run it inside a
  `tmux` / `zellij` pane with scrollback. Not a production process
  manager — for that, the answer is the platform's native one
  (systemd / launchd / Kubernetes / ECS).
