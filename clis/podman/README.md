# podman

- **Repo:** https://github.com/containers/podman
- **Version:** 5.8.2 (tagged 2026-04-14; latest stable on the 5.x line; 6.x development is on `main` but not yet tagged GA)
- **License:** Apache-2.0 — see [`LICENSE`](https://github.com/containers/podman/blob/main/LICENSE)
- **Language:** Go (CLI + libpod) with a thin Rust helper (`netavark` / `aardvark-dns` for the rootless network stack)
- **Install:** `brew install podman` · `apt install podman` · `dnf install podman` · `pacman -S podman` · or download a static binary from the GitHub release page (`podman-remote-static-linux_amd64.tar.gz`, `podman-installer-macos-arm64.pkg`, `podman-v5.8.2-setup.exe`); macOS / Windows additionally need a Linux VM, supplied by `podman machine init && podman machine start` which wraps `qemu` (macOS / Linux) or `wsl2` (Windows)

## Overview

`podman` is a daemonless, OCI-compliant container engine
that exposes a CLI surface that is **command-for-command
compatible** with the Docker CLI (`alias docker=podman`
just works for the 95% case). Each `podman run` forks a
short-lived `conmon` monitor process directly under the
invoking user; there is no long-running root daemon
holding open all containers. By default it runs
**rootless**: containers execute inside a user namespace
mapped via `/etc/subuid` + `/etc/subgid`, so a container
that thinks it is `root:0` is actually running as the
unprivileged calling user on the host, which means a
container escape lands you as a normal user, not root.
Storage uses `containers/storage` with `overlay` (kernel
overlayfs in rootful mode, `fuse-overlayfs` in rootless),
images come from `containers/image` (so `podman pull` /
`buildah build` / `skopeo copy` all share one
registry + signature + auth stack), and the network stack
uses `netavark` + `aardvark-dns` (CNI is deprecated as of
4.x). The killer differentiator is `podman generate
kube` / `podman play kube`: a running pod (yes, podman
groups containers into pods natively, the CNCF kind not
the Docker kind) can be serialised to a Kubernetes YAML
manifest, edited, and replayed on any cluster — the same
artifact runs on a laptop and in production. `podman
machine` brings a Linux-VM-managed engine to macOS and
Windows so the same CLI works everywhere.

## Niche

**Daemonless, rootless-by-default, Docker-CLI-compatible
OCI container engine that emits Kubernetes YAML as its
native serialisation format.** The role is "the
container engine you point your dev loop at when you
care about (a) not running a root daemon, (b) producing
artifacts a real cluster will accept, or (c) running on
RHEL / Fedora / hardened-Ubuntu where Docker is not the
shipping default". The competing universe is `docker` /
`nerdctl` / `colima` / `lima` / `containerd` / `crun` /
`runc` — see comparisons below.

## When to use

- You want a Docker-shaped CLI without a root-owned
  always-on daemon: `podman run -d --name web nginx`
  produces a `conmon` process owned by your user, no
  `dockerd` socket to harden, no group membership that
  is effectively root.
- You are on RHEL / Fedora / CentOS Stream / Rocky / Alma
  where podman is the shipping default and `docker` is
  the third-party install — every podman feature is
  tested first on those distros.
- You want **rootless containers as the default**: no
  `--user 1000:1000`, no entrypoint juggling, no
  read-only-root-fs workarounds; the container is
  unprivileged from the kernel's point of view.
- You want to develop locally and ship to Kubernetes
  without rewriting the manifest: `podman play kube
  pod.yaml` runs the same YAML `kubectl apply -f
  pod.yaml` would.
- You want pods (multiple containers sharing a network
  namespace + IPC + UTS) as a first-class CLI primitive:
  `podman pod create --name web` then `podman run --pod
  web ...` mirrors the Kubernetes pod model exactly.
- You are on macOS and want a `brew install`-able
  container engine that plays well with Apple Silicon:
  `podman machine init --rootful=false` brings up a
  fedora-coreos VM under qemu and the CLI talks to it
  over an SSH-tunnelled REST socket.

## When NOT to use

- Your toolchain hard-codes the Docker socket path
  (`/var/run/docker.sock`) and the consuming code does
  not honour `DOCKER_HOST` — you can `systemd --user
  enable podman.socket` to expose a Docker-API-shaped
  socket and `export DOCKER_HOST=unix:///run/user/1000/podman/podman.sock`
  but legacy tools that `stat()` the literal path break.
  Use `docker` directly or run podman in compat mode.
- You need **Docker Compose v2** with full feature parity
  including BuildKit-cache-mount semantics — `podman
  compose` shells out to `docker-compose` or
  `podman-compose` and the latter trails Compose v2 on
  niche keys. Use `docker compose` if Compose is the
  source of truth.
- You need **Docker Desktop's GUI + Kubernetes
  integration + virtiofs file-sharing on macOS**;
  Podman Desktop exists but is younger and the
  file-sharing story is `gvisor-tap-vsock` instead of
  `virtiofs`.
- You need an engine that is **Swarm-mode compatible**
  (`docker service create`, `docker stack deploy`) —
  podman has no Swarm; the upgrade path is
  `play kube` → real Kubernetes (k3d / k3s / kind /
  microk8s).

## Comparison vs alternatives in zoo

- [`colima`](../colima/) — Lima-based VM manager that
  runs `containerd` + `nerdctl` (or Docker / k3s) on
  macOS. Pick `colima` when you want `nerdctl`
  semantics, BuildKit-native, and Lima's vmnet
  networking; pick `podman` when you want the rootless
  Docker-CLI compat layer + `play kube` workflow + the
  exact same engine you would run on RHEL in
  production.
- [`nerdctl`](../nerdctl/) — Docker-shaped CLI on top of
  `containerd`. Closer to the Kubernetes runtime
  (`containerd` is what kubelet talks to) and ships
  BuildKit, ipfs registry, and rootless containerd
  out of the box. Pick `nerdctl` when "use the same
  runtime kubelet uses" is the goal; pick `podman`
  when you want pods + `play kube` + the systemd
  generator (`podman generate systemd` → `podman
  quadlet`) for hand-managed hosts.
- [`lima`](../lima/) — Linux-VM-on-macOS substrate; both
  `colima` and standalone setups use it. Orthogonal —
  podman uses its own `podman machine` (qemu-based, not
  Lima) on macOS, so they do not compete directly.
- [`apko`](../apko/) + [`melange`](../melange/) +
  [`ko`](../ko/) — declarative OCI-image builders.
  Complementary — `apko publish` produces an OCI image
  that `podman pull` / `podman run` consumes; podman
  is the runtime, apko is the builder.
- [`crane`](../crane/) / [`regctl`](../regctl/) /
  [`oras`](../oras/) — registry clients. Complementary —
  `podman push` does the same job but those tools cover
  the read-only / inspection / multi-arch-manifest
  surface more thoroughly without needing a local
  storage graph.
- [`dive`](../dive/) — image-layer inspector. Pure
  complement — point `dive podman://localhost/myimage`
  at any image podman has pulled.

## Why it earns a slot in an AI-native workflow

Coding agents that fan out to "run this generated code
in a sandbox" land squarely on the
container-without-a-root-daemon use case: an agent
running on a laptop should not require `sudo
systemctl start docker` to spawn a sandbox, should not
hand the agent a socket whose group membership is
root-equivalent, and should not need a separate
identity for "agent calling docker" vs "user calling
docker". `podman run --rm --rootfs ./sandbox` from an
agent process inherits the agent's UID, gets a kernel
user-namespace mapping for free, and dies cleanly when
the agent exits. The `podman generate kube` path also
matters: an agent that builds a multi-container
artifact (model server + vector store + small worker)
can emit a single `kube.yaml` the human reviews and
ships to a real cluster, without the agent ever having
to learn about Helm charts. `podman quadlet` (systemd
unit-of-truth for containers) plus `podman auto-update`
covers the "agent-managed long-running service on the
laptop" case without hand-rolling launchd plists.

## Example invocations

```bash
# Quick sanity check — docker users feel at home
podman run --rm hello-world
alias docker=podman    # the 95% case

# Rootless by default — confirm the namespace
podman run --rm alpine id
# uid=0(root) gid=0(root) ... but on the host:
podman top $(podman run -d alpine sleep 100) huser hpid
# huser is your shell user, not root

# Build, run, and serialise a small pod
podman pod create --name web -p 8080:80
podman run -d --pod web --name nginx nginx
podman run -d --pod web --name redis redis
podman generate kube web > web-pod.yaml
# web-pod.yaml is a valid Kubernetes manifest — kubectl apply -f web-pod.yaml works

# Replay a Kubernetes manifest locally
podman play kube web-pod.yaml

# macOS / Windows: bring up the Linux VM that hosts the engine
podman machine init --cpus 4 --memory 8192 --disk-size 60
podman machine start
podman info | grep -A1 "host:"

# Quadlet — systemd unit as the source of truth for a container
mkdir -p ~/.config/containers/systemd
cat > ~/.config/containers/systemd/redis.container <<'EOF'
[Container]
Image=docker.io/library/redis:7
PublishPort=6379:6379

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user start redis.service

# Auto-update on a tag pull (rolling-deploy style)
podman run -d --label "io.containers.autoupdate=registry" \
  --name app ghcr.io/me/app:latest
systemctl --user enable --now podman-auto-update.timer

# Expose Docker-API-shaped socket for legacy tools
systemctl --user enable --now podman.socket
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
docker-compose up -d   # talks to podman, not dockerd
```
