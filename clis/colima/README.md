# colima

## What it does
Container runtimes on macOS (and Linux) in a single Go binary built on top of [Lima](../lima/). `colima start` boots a lightweight VM with `containerd` + `nerdctl` (or the Docker engine if you pass `--runtime docker`), wires the Docker socket through to the host, registers a kubectl-ready k3s cluster on demand (`--kubernetes`), and exposes Rosetta-accelerated x86_64 emulation for amd64 images on Apple Silicon (`--vm-type vz --vz-rosetta`). One command replaces the proprietary Docker desktop install path entirely; `colima stop` and `colima delete` do exactly what they say with no leftover services or trayicon.

## Why it's interesting
The macOS local-container story has fragmented: the popular paid desktop app, OrbStack, Rancher Desktop, raw `lima` + `nerdctl`, podman-machine. `colima` is the smallest opinionated wrapper that gets a `docker` and `kubectl` working on a fresh laptop in two commands (`brew install colima docker && colima start`) without requiring a tray app, a license server, or hand-editing a Lima YAML. Profiles (`colima start --profile gpu --cpu 8 --memory 16 --disk 100`) let one host run several isolated VMs with different runtimes side-by-side; `colima ssh` drops into the VM for debugging; `colima nerdctl install` symlinks `nerdctl` so containerd-native `nerdctl build` / `nerdctl compose` Just Work without Docker. The v0.10 line added a `colima model` subcommand that runs local LLM model containers via Docker Model Runner or Ramalama for the AI-on-laptop crowd, but the core value is still "Docker without Docker Desktop, in one Go binary."

## Niche category
Container runtime VM manager for macOS / Linux — Docker-socket-compatible local container host without a desktop app

## Repo
https://github.com/abiosoft/colima

## Version pinned
`v0.10.1`

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux) — most common path
brew install colima docker

# MacPorts
sudo port install colima

# Pre-built binary (Linux x86_64)
curl -LO https://github.com/abiosoft/colima/releases/download/v0.10.1/colima-Linux-x86_64
sudo install colima-Linux-x86_64 /usr/local/bin/colima

# From source (requires Go 1.22+)
go install github.com/abiosoft/colima@v0.10.1
```

## Usage
```sh
# Start with sensible defaults (containerd runtime, 2 CPU / 2 GiB / 60 GiB disk)
colima start

# Or start with the Docker engine + bigger VM + Apple Silicon Rosetta amd64 emulation
colima start --runtime docker --cpu 6 --memory 12 --disk 80 --vm-type vz --vz-rosetta

# Bundle a local k3s cluster — kubectl context auto-registered
colima start --kubernetes
kubectl get nodes

# Use named profiles for parallel VMs
colima start --profile dev   --cpu 4 --memory 8
colima start --profile build --cpu 8 --memory 16
colima list

# Drive containers — the host docker / nerdctl CLI talks to the VM transparently
docker run --rm hello-world
nerdctl run --rm -it alpine sh

# Drop into the VM for debugging
colima ssh

# Stop / delete cleanly (no orphaned launchd jobs, no tray icon)
colima stop
colima delete --force
```

## When to pick `colima` vs alternatives
- **vs Docker Desktop**: colima is the licence-free, no-tray, single-binary alternative. Pick Docker Desktop only when you need its GUI dashboard, the bundled Compose UI, or its first-party Kubernetes UI; pick colima otherwise.
- **vs raw [`lima`](../lima/) + [`nerdctl`](../nerdctl/)**: lima is the lower-level VM manager colima is built on. Reach for lima directly when you need a non-container Linux VM (Ubuntu / Fedora / Arch images), custom cloud-init, or unusual networking; pick colima when you specifically want "Docker socket on the host".
- **vs OrbStack**: OrbStack is faster and more polished on macOS but is closed-source freemium for personal / paid for commercial. Pick colima when you need a permissively-licensed open-source option that runs anywhere lima runs.
- **vs Rancher Desktop**: Rancher Desktop bundles a UI, k3s, and a tray app and skews toward Kubernetes-first workflows. Pick colima when you want CLI-only and the smallest moving-parts surface.
- **vs `podman machine`**: podman machine is the equivalent in the Podman ecosystem. Pick podman when you specifically want rootless / daemonless / pod-grouping semantics; pick colima when the rest of your tooling assumes a Docker socket.
