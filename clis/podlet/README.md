# podlet

> **Generates Podman Quadlet unit files (and Kubernetes /
> Compose specs) from existing `podman` commands or running
> containers.** Bridges the gap between "I have a working
> `podman run …`" and "I want this declared as a systemd-managed
> Quadlet under `~/.config/containers/systemd/`". Pinned to
> **v0.3.1**
> ([LICENSE](https://github.com/containers/podlet/blob/main/LICENSE),
> MPL-2.0; version checked via
> `gh release view --repo containers/podlet`).

Source: <https://github.com/containers/podlet>

## TL;DR

Quadlet is the modern way to run Podman containers under
systemd: you drop a `.container` / `.pod` / `.network` /
`.volume` / `.kube` file into the systemd unit search path and
`systemctl --user daemon-reload` materialises a real systemd
service that pulls, runs, restarts, and logs the container.
Authoring those unit files by hand is tedious. `podlet` reads a
`podman run` / `podman pod create` / `podman network create`
invocation (or introspects a live container by ID) and emits
the equivalent Quadlet file. It can also convert Compose YAML
into the same target, so a docker-compose-shaped repo can move
to systemd-supervised, rootless Podman without rewriting the
service definitions.

## Install

```bash
# Cargo
cargo install --locked podlet

# Single-binary release
curl -L https://github.com/containers/podlet/releases/download/v0.3.1/podlet-x86_64-unknown-linux-gnu.tar.xz \
  | tar xJ && sudo mv podlet /usr/local/bin/

# Container (run podlet itself in podman)
podman run --rm quay.io/containers/podlet:v0.3.1 \
  podman run nginx
```

## Example

```bash
# Convert a podman command into a .container Quadlet
podlet podman run -d --name caddy -p 8080:80 caddy:2

# Introspect a running container and emit its Quadlet
podlet generate container caddy

# Convert a Compose file (writes one unit per service)
podlet compose --pod web ./compose.yaml

# Emit Kubernetes YAML for the .kube Quadlet flavour
podlet --kube podman run -d nginx

# Write directly into the systemd user search path
podlet --file ~/.config/containers/systemd/ \
  podman run -d --name redis redis:7
```

## When to use

- You're moving from Docker Compose to rootless Podman +
  systemd and want a mechanical translation rather than a
  hand-rewrite.
- You prototype with `podman run` on the CLI and want to
  promote a working command into a supervised, restart-on-boot
  service without reverse-engineering the Quadlet schema.
- You manage a fleet of single-host Podman boxes (edge, CI
  runners, home labs) where Kubernetes is overkill but you
  still want declarative, version-controlled service files.

## When NOT to use

- You're on Docker, not Podman — Quadlet is a Podman+systemd
  feature; reach for `docker compose`, `nerdctl compose`, or
  the [`compose-spec`](https://compose-spec.io) tooling instead.
- You're targeting Kubernetes for real — generate manifests
  with [`kompose`](../kompose/) or write them directly; podlet's
  `--kube` mode is for the *Quadlet* `.kube` flavour, not for
  cluster deployment.
- You need rolling updates, secrets management, or multi-node
  scheduling — that's the Kubernetes / Nomad layer, not
  systemd-supervised Podman.

## Orthogonality vs existing zoo entries

- **vs [`kompose`](../kompose/)** — kompose translates Compose
  → Kubernetes manifests for cluster deployment; podlet
  translates Compose / `podman run` → systemd Quadlet units
  for single-host supervised containers. Different targets.
- **vs [`podman`](https://podman.io)** — podman *runs*
  containers; podlet *describes* them as systemd units.
- **vs [`nerdctl`](../nerdctl/)** — nerdctl is a Docker-CLI
  drop-in over containerd; podlet is Podman-specific tooling
  for the systemd integration story.
- **vs [`process-compose`](../process-compose/)** — both
  supervise long-running processes on one host;
  process-compose is its own scheduler with its own YAML,
  podlet emits artefacts the OS scheduler (systemd) already
  understands.

## Caveats

- Quadlet itself requires Podman 4.4+ and systemd 250+; older
  RHEL / Debian releases need a backport.
- The Compose → Quadlet translation covers the common subset
  (services, networks, volumes, env, healthchecks); exotic
  Compose features (`extends`, `configs`, profiles) may need
  manual touch-up. Run `podlet compose` and review the diff
  before committing.
- Generated unit files reference image tags as written; pin to
  digests (`image@sha256:…`) before deploying to production
  hosts to avoid silent re-pulls.
