# lazydocker

> **A terminal UI for Docker and Docker Compose** — a single Go
> binary that renders containers, images, volumes, networks, and
> compose services as a navigable TUI, with logs, stats, env, and
> config one keystroke away. Pinned to **v0.25.2** (commit
> `7e7aadc2071d58031bf2daafca1fbd4093efc23f`,
> [LICENSE](https://github.com/jesseduffield/lazydocker/blob/master/LICENSE),
> MIT).

Source: <https://github.com/jesseduffield/lazydocker>

## TL;DR

`lazydocker` is what you reach for when `docker ps`, `docker logs
-f`, `docker stats`, `docker inspect`, and `docker compose
restart` would all be the next four commands you type. It's a
single ~20 MB binary with no daemon of its own — it talks to your
local Docker socket (or a remote `DOCKER_HOST`) and renders five
panes: project (compose services), containers, images, volumes,
networks. Arrow keys move between rows, `enter` opens logs,
`s` shows live stats, `e` shows env, `r` restarts, `d` removes,
`x` opens a per-resource menu of "things you'd otherwise google".
Everything you can do is a single key, and the help bar at the
bottom always tells you what those keys are right now.

## Install

```bash
# Homebrew (macOS / Linux)
brew install lazydocker

# Linux package managers
# Arch:   pacman -S lazydocker
# Nix:    nix-env -iA nixpkgs.lazydocker
# Scoop:  scoop install lazydocker

# Go install (any OS with a Go toolchain)
go install github.com/jesseduffield/lazydocker@latest

# from a release tarball (any OS)
curl -Lo lazydocker.tar.gz "https://github.com/jesseduffield/lazydocker/releases/download/v0.25.2/lazydocker_0.25.2_Darwin_arm64.tar.gz"
tar xf lazydocker.tar.gz
sudo install lazydocker /usr/local/bin/

# verify
lazydocker --version    # Version: 0.25.2

# launch (no flags needed; uses $DOCKER_HOST or /var/run/docker.sock)
lazydocker
```

Optional: drop a `~/.config/lazydocker/config.yml` to remap
keys, change the theme, or pin a custom command runner (e.g.
`podman` instead of `docker`).

## License

MIT — see
[LICENSE](https://github.com/jesseduffield/lazydocker/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. boot the TUI against the local socket
lazydocker

# inside the TUI:
#   ↑/↓                move row
#   tab / [ / ]        switch pane (project / containers / images / volumes / networks)
#   enter              open logs (streams live; / to search, n / N to step matches)
#   s                  live stats (CPU%, mem, net I/O, block I/O)
#   e                  env vars on this container
#   c                  config (the merged compose / inspect view)
#   t                  top (process list inside the container)
#   r                  restart           d        remove (with prompt)
#   p                  pause / unpause   m        view dockerfile
#   R                  recreate          (project pane) U   compose up   D  compose down
#   x                  per-row menu of every action

# 2. point at a remote daemon (TLS or SSH)
DOCKER_HOST=ssh://user@build-box.internal lazydocker

# 3. keep it open in a tmux pane while you iterate on a compose file
docker compose up -d                # in one pane
lazydocker                          # in another — `R` recreates after each edit

# 4. clean up the cruft (the panel that surprises everyone the first time)
# images pane → x → "prune unused images"
# volumes pane → x → "prune unused volumes"
# containers pane → x → "remove all stopped containers"

# 5. follow logs of a single compose service without leaving the TUI
# project pane → select service → enter → / "ERROR" to scope view
```

## Niche It Fills

**Docker as a TUI, not a flag salad.** The Docker CLI is fine
when you know exactly which container ID and which subcommand,
but the moment you're in "what's the state of everything right
now and which one is the noisy one" mode, you end up alternating
`docker ps`, `docker logs -f <prefix>`, `docker stats`, and
`docker compose restart <svc>` for 20 minutes. `lazydocker`
collapses that loop into a 5-pane dashboard with one-key actions
and live tailing — same primitives, ~5× fewer keystrokes per
investigation.

## Why use it

Three things `lazydocker` does that the Docker CLI does not:

1. **All five resource types in one view, with cross-links.**
   Project → containers → images → volumes → networks are panes
   you switch with `tab`. Selecting a compose service highlights
   the containers it owns; selecting a container highlights the
   image and volumes it uses. No `docker inspect | jq` to figure
   out "which volume is this service writing to".
2. **Live logs + live stats without juggling terminals.** `enter`
   on a container opens a follow-tailing log pane that's
   searchable in-place; `s` swaps to live CPU / mem / I/O. You
   stay in one terminal and one keymap instead of one tmux split
   per `docker logs -f` invocation.
3. **Compose-aware, not just Docker-aware.** The project pane
   understands `docker-compose.yml` services, lets you `U` (up),
   `D` (down), `R` (recreate), or restart per-service from the
   keyboard. It works against `docker compose` v2 plugins out of
   the box and against `podman-compose` with a one-line config
   override.

For agent / LLM workflows where the model is iterating on a
container build (`Dockerfile` edits, `compose up`, watch logs
for a stack trace, fix, repeat), keeping `lazydocker` open in a
side pane gives the human reviewer a constant-time read on
"what's actually running and what's it saying" without polling
`docker ps` between steps.

## Vs Already Cataloged

- **Vs [`lazygit`](../lazygit/):** sibling project from the same
  author (Jesse Duffield) using the same TUI library (`gocui`).
  `lazygit` is to `git` what `lazydocker` is to `docker`: the
  five-pane keyboard-first view of all the moving parts. Same
  muscle memory transfers — `?` for help, `x` for menus, `enter`
  to drill in. Most users install both together.
- **Vs `docker` CLI directly:** `docker` wins for scripted /
  automated flows (CI, Makefiles, `docker compose run --rm` one-
  shots) where you want each command to be reproducible and
  exit-code-friendly. `lazydocker` wins for interactive
  investigation where you don't yet know which container, which
  log line, or which compose service is at fault.
- **Vs Docker Desktop's GUI dashboard:** Docker Desktop renders
  the same data as a graphical app, but it's heavier (Electron-
  shaped), needs the Docker Desktop license for some org sizes,
  and isn't usable over SSH. `lazydocker` is one binary, runs
  inside `tmux` over SSH to a build box, and has zero licensing
  ambiguity.

## Caveats

- **Read-mostly safety, write-actions are one keystroke.** `d`
  on a container removes it (with a confirm prompt, but still
  one extra keystroke). `x` → "prune everything" is genuinely
  destructive. Don't hand the keyboard to someone who hasn't
  seen the help bar.
- **Compose v1 vs v2 detection.** It tries `docker compose`
  (v2 plugin) first, falls back to `docker-compose` (v1
  Python). If both are installed and disagree about project
  state you can get stale views; pin one in
  `~/.config/lazydocker/config.yml` under `commandTemplates`.
- **No multi-host / swarm / k8s view.** It's a single-daemon
  tool — one `DOCKER_HOST` at a time. For Kubernetes use
  [`k8sgpt`](../k8sgpt/) for analysis or `k9s` (not cataloged)
  for the equivalent TUI; `lazydocker` plus `k9s` is the common
  pair.
- **Log buffer is in-memory and bounded.** The follow-tail pane
  caps at a configurable line count (default ~1000); for full
  history use `docker logs <id>` or stream to a file. Don't
  rely on `lazydocker`'s buffer for incident forensics.
- **Project-pane state cache.** It caches compose-project
  membership for ~5 s for performance; if you `docker compose up`
  in another terminal, the new services may not appear until the
  next refresh tick or until you press `R` in the project pane.
