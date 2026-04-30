# lazydocker

> **Terminal UI for Docker and Docker Compose.** Single-binary
> TUI that wraps the docker / compose CLI behind a keyboard-
> driven dashboard: containers, services, images, volumes, and
> networks in panes you can tab through, with live logs, stats
> graphs, env / config inspection, and one-key restart / stop /
> remove / prune actions. Pinned to **v0.25.2**
> ([LICENSE](https://github.com/jesseduffield/lazydocker/blob/master/LICENSE),
> MIT).

Source: <https://github.com/jesseduffield/lazydocker>

## TL;DR

Run `lazydocker` in a project dir and the right pane shows the
selected container's logs streaming live, CPU / memory sparkline,
env vars, and the exact `docker inspect` JSON; left pane lists
containers (or compose services if a `docker-compose.yml` is
present). Hotkeys cover the 90 % path: `r` restart, `s` stop,
`d` remove, `e` exec into shell, `b` open bulk-action menu
(prune images / volumes / networks / build cache). Custom
commands and per-container menus are wired via a YAML config so
"shell into postgres + run a query" becomes one keystroke.

## Install

```bash
# Homebrew (macOS / Linux)
brew install lazydocker

# Install script
curl https://raw.githubusercontent.com/jesseduffield/lazydocker/master/scripts/install_update_linux.sh \
  | bash

# Go install
go install github.com/jesseduffield/lazydocker@latest

# Docker (mount the host docker socket)
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/.config/jesseduffield/lazydocker:/.config/jesseduffield/lazydocker \
  lazyteam/lazydocker
```

## Example

```bash
# Launch against the current project (auto-detects compose file)
lazydocker

# Tail a specific service's logs from a compose project elsewhere
lazydocker --project-dir /srv/stack
```

## When to use

- You spend the day toggling between `docker ps`, `docker logs
  -f`, `docker stats`, and `docker compose restart` and want
  one keyboard surface for all of it.
- You manage a multi-service compose stack locally and need to
  bounce / inspect individual services without remembering
  service names.
- You want to prune dangling images / volumes / build cache
  interactively rather than memorising `docker system prune`
  flag combinations.

## When NOT to use

- You manage Kubernetes, not Docker — use `k9s` instead.
- You need scripted, non-interactive automation; this is a TUI,
  not a headless tool.
- The host has no Docker socket access (rootless / remote-only
  cluster setups will not surface containers correctly).
