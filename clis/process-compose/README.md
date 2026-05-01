# process-compose

## What it does
Docker-Compose-style orchestrator for plain OS processes: one `process-compose.yaml` declares processes with `command`, `depends_on`, readiness probes (`liveness_probe` / `readiness_probe`), restart policies, log routing, and TUI ordering, and `process-compose up` runs the lot in the foreground with a curses-style TUI showing live status, logs, and per-process start/stop/restart controls. No containers, no daemon — just a supervised process tree.

## Why it's interesting
Fills the gap between "a `Procfile` and `foreman`" and "spin up docker-compose for local dev": you get health checks, dependency ordering, and a real TUI without paying the container-runtime tax, which matters on macOS dev loops where Docker Desktop is the slowest part of the inner loop. Works as a drop-in for devshell stacks (paired with nix / devbox / mise) where the components are native binaries and containerizing them just to orchestrate them is overkill.

## Niche category
Process orchestration — local multi-process supervisor with TUI

## Repo
https://github.com/F1bonacc1/process-compose

## Version pinned
`v1.103.0`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
brew install f1bonacc1/tap/process-compose
# or
curl -fsSL https://github.com/F1bonacc1/process-compose/releases/latest/download/process-compose_$(uname -s)_$(uname -m).tar.gz | tar -xz -C /usr/local/bin process-compose
```

## Usage examples
```sh
# Start everything declared in ./process-compose.yaml with the TUI
process-compose up

# Headless mode for CI, with structured JSON logs
process-compose up --tui=false --log-level=info

# Attach the TUI to a process-compose instance started elsewhere
process-compose attach
```

Minimal config:
```yaml
# process-compose.yaml
processes:
  api:
    command: "go run ./cmd/api"
    readiness_probe:
      http_get: { host: 127.0.0.1, port: 8080, path: /healthz }
  worker:
    command: "go run ./cmd/worker"
    depends_on:
      api: { condition: process_healthy }
```
