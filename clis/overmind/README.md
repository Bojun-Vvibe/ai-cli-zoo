# overmind

> **A `Procfile`-based process manager for development that
> runs each declared process inside a `tmux` window so every
> child gets a real PTY (interactive `binding.pry` / `pdb` /
> `byebug` debuggers actually work), supports per-process
> restart without killing the whole stack (`overmind restart
> web`), connection into a single process's output
> (`overmind connect web` attaches to that tmux window),
> environment overlay via `.overmind.env`, formation control
> (`OVERMIND_FORMATION=web=2,worker=3`), and graceful SIGTERM
> propagation that `foreman` famously gets wrong.** Pinned to
> **v2.5.1** (released 2026-03-26),
> [LICENSE](https://github.com/DarthSim/overmind/blob/v2.5.1/LICENSE),
> MIT.

Source: <https://github.com/DarthSim/overmind>

## TL;DR

`overmind` reads a standard `Procfile` (the same format
`foreman` / Heroku introduced) and runs each line as a
managed process. The trick: instead of forking children
directly and multiplexing their stdout into one terminal
(the foreman model, which loses TTY semantics and makes
debuggers unusable), overmind starts a detached `tmux`
session and gives each process its own window. That single
choice unlocks:

- **Real PTYs** — `binding.pry`, `ipdb.set_trace()`,
  `delve`, `node --inspect-brk`, REPLs, anything that needs
  `isatty(stdin)` to behave.
- **Per-process restart** — `overmind restart worker` cycles
  *only* that window; the web server, asset pipeline, and
  database clients keep running.
- **Per-process attach** — `overmind connect web` drops you
  into that process's tmux window with full scrollback;
  detach with `Ctrl-b d` and the process keeps running.
- **Graceful shutdown** — SIGTERM to overmind fans out to
  every window with the correct signal and a configurable
  timeout (`-t 30`) before SIGKILL, which matches what
  Kubernetes / systemd expect.

A `Procfile` looks like this:

```procfile
web:     bundle exec puma -C config/puma.rb
worker:  bundle exec sidekiq -C config/sidekiq.yml
assets:  bin/vite dev
db:      postgres -D /usr/local/var/postgres
redis:   redis-server
```

`overmind start` brings the whole stack up; `overmind quit`
brings it down cleanly.

## Install

```bash
# Homebrew (macOS / Linux) — pulls tmux as dep
brew install overmind tmux

# Go install (any platform with Go ≥ 1.22)
go install github.com/DarthSim/overmind/v2@v2.5.1

# Static binary from GitHub Releases (linux / macOS / freebsd, amd64 / arm64)
curl -fsSL -o /tmp/overmind.gz \
  https://github.com/DarthSim/overmind/releases/download/v2.5.1/overmind-v2.5.1-linux-amd64.gz
gunzip /tmp/overmind.gz && chmod +x /tmp/overmind
sudo install /tmp/overmind /usr/local/bin/overmind

# Verify
overmind --version    # Overmind version 2.5.1
tmux -V               # required runtime dep
```

## One Concrete Example

```bash
# Procfile.dev — typical Rails + Sidekiq + Vite stack
cat > Procfile.dev <<'EOF'
web:     bin/rails server -p 3000
worker:  bundle exec sidekiq -C config/sidekiq.yml
vite:    bin/vite dev
css:     bin/rails tailwindcss:watch
EOF

# .overmind.env — per-developer overrides, gitignored
cat > .overmind.env <<'EOF'
PORT=3000
RAILS_ENV=development
DATABASE_URL=postgres://localhost/myapp_dev
EOF

# Start the stack (foreground, all output prefixed by process name)
overmind start -f Procfile.dev

# In another terminal: jump into the web process for a debugger session
overmind connect web
# (you're now inside tmux attached to bin/rails server.
#  hit your binding.pry; type your way out; Ctrl-b d to detach.)

# Bounce only the worker after a Sidekiq config change
overmind restart worker

# Run two web instances + one worker
OVERMIND_FORMATION=web=2,worker=1 overmind start -f Procfile.dev

# Echo a command into a process's stdin (great for REPLs)
overmind echo "User.count" web

# Stop everything cleanly
overmind quit
```

## License

[MIT](https://github.com/DarthSim/overmind/blob/v2.5.1/LICENSE),
SPDX `MIT`.

## Niche / positioning

The **PTY-correct successor to `foreman`** for local dev
process orchestration. Pick `overmind` over
[`hivemind`](https://github.com/DarthSim/hivemind) (same
author, no tmux dependency) when you need per-process
restart, attach, and debugger PTYs — those are the entire
reason overmind exists. Pick `hivemind` when tmux isn't
available (CI, minimal containers) and you only need basic
multiplex. Pick [`mprocs`](../mprocs/) when you want a TUI
dashboard with mouse / keyboard navigation across processes
in one window, but you don't need the per-window detachable
tmux model. Skip overmind in production — it is explicitly a
*development* tool; production uses systemd, Kubernetes,
Nomad, or your platform's process supervisor. Skip when
your stack is already Docker Compose — `compose up` covers
the same Procfile-shaped need with container isolation.
