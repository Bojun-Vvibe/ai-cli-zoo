# tilt

> **Multi-service local development for Kubernetes and Docker
> Compose.** Reads a `Tiltfile` (Starlark) describing how to
> build and deploy your services, then runs a watch loop:
> source change → image rebuild (or in-place file sync) →
> redeploy → live log + status in a web UI / TUI. Optimises
> the inner loop with `live_update` (rsync-then-restart inside
> the running container, no rebuild) and the `tilt-dev/ctlptl`
> companion for ephemeral local clusters. Pinned to **v0.37.2**
> ([LICENSE](https://github.com/tilt-dev/tilt/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/tilt-dev/tilt>

## TL;DR

`tilt up` reads `./Tiltfile` and stands up every resource it
declares — `docker_build` / `custom_build` produces images,
`k8s_yaml` / `helm` / `kustomize` deploys them, `local_resource`
runs side-band scripts (lint, codegen, test). The web UI on
`localhost:10350` shows per-resource status, build logs, pod
logs, and triggers; `tilt down` tears the stack back down.
`live_update` declarations turn "edit a Python file" into a
sub-second sync into the running pod instead of a 30-second
image rebuild — the central performance trick that
distinguishes Tilt from plain `skaffold dev`.

## Install

```bash
# Homebrew (macOS / Linux)
brew install tilt-dev/tap/tilt

# Install script
curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh \
  | bash

# Go install
go install github.com/tilt-dev/tilt/cmd/tilt@latest
```

## Example

A minimal `Tiltfile`:

```python
# Build the image from ./api, with live-sync of *.py files
docker_build(
  'registry.local/api',
  context='./api',
  live_update=[
    sync('./api', '/app'),
    run('pip install -r requirements.txt', trigger=['./api/requirements.txt']),
    restart_container(),
  ],
)

# Apply the rendered Helm chart
k8s_yaml(helm('./charts/api', name='api'))
k8s_resource('api', port_forwards='8080:8080')

# Side-band: run unit tests on every change to ./api
local_resource(
  'api-tests',
  cmd='pytest -q api/tests',
  deps=['./api'],
  resource_deps=['api'],
)
```

```bash
tilt up        # interactive dev loop
tilt ci        # one-shot, exits non-zero on any failure (CI mode)
tilt down      # tear down everything Tilt created
```

## When to use

- You develop a multi-service app (frontend + several backend
  services + a job worker) and want one command to bring the
  whole stack up against a local cluster.
- Your inner-loop pain is "image rebuild + push + redeploy on
  every save"; `live_update` collapses that to a file sync.
- You want a UI showing per-service status, logs, and trigger
  buttons so the rest of the team can drive the dev stack
  without learning every kubectl invocation.

## When NOT to use

- Single-service projects where `docker compose up` or `go
  run` is enough — Tilt's value is multi-service
  orchestration.
- You need production deployment tooling — Tilt is for the
  dev loop; pair it with Helm / ArgoCD / Flux for prod.
- You cannot run a local Kubernetes (kind / k3d / minikube)
  or a remote dev cluster; Tilt assumes a real cluster
  endpoint.
