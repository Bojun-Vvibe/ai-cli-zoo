# distrobox

## What it does
A POSIX-shell wrapper around `podman`, `docker`, or `lilipod` that creates
**tightly host-integrated Linux containers** you treat as alternate userland
distributions. `distrobox-create --image fedora:40 --name fed` boots a
long-running rootless container that bind-mounts your `$HOME`, `/tmp`,
`/dev`, the host SSH agent, the Wayland/X11 socket, the host GPU devices,
PulseAudio/PipeWire, and the host user's UID/GID, so `distrobox enter fed`
drops you in a shell where `code .`, `nvim`, `firefox`, and CUDA all work
against your real files. `distrobox-export --app firefox` writes a `.desktop`
file on the host that launches the containerized binary as if it were native.
Behind the scenes it is just one shell script + a generated `entrypoint.sh`
that runs as PID 1 inside the container; there is no daemon, no kernel
module, no custom runtime.

## Why it's interesting
Different shape from `toolbox` (Fedora-only, drops several of the host
mounts), `nix-shell` (recreates dependencies but keeps the host kernel
userland — no apt, no dnf), `docker run -it` (clean but unintegrated — your
files, GPU, audio, and display are not there unless you wire each one), and
`devcontainer` (project-scoped, VS Code-driven, ephemeral). distrobox is the
"give me a persistent Arch / Ubuntu / openSUSE userland on top of my
immutable host" tool — it is the standard answer on Fedora Silverblue, SteamOS,
Bluefin, Bazzite, and any other ostree / image-based desktop where you cannot
`dnf install` arbitrary things on the host. Choose it when you want
`apt install` *and* GPU *and* your home directory in the same shell; do
**not** choose it for production container workloads (use `podman` /
`docker` directly) or for hermetic reproducible builds (use `nix` or
`devcontainer`).

## Niche category
Host-integrated mutable Linux container — alternate userland on top of
immutable / ostree desktops.

## Repo
https://github.com/89luca89/distrobox

## Version pinned
`1.8.1.2` (latest tagged release, `1.8.1.2`)

## License
- SPDX: `GPL-3.0-only`
- License file in upstream repo: `COPYING.md`

## Install
```sh
# Upstream one-liner (writes to ~/.local; no root needed)
curl -s https://raw.githubusercontent.com/89luca89/distrobox/main/install | sh -s -- --prefix ~/.local

# Or via package managers
brew install distrobox             # macOS / Linuxbrew (needs podman/docker)
sudo dnf install distrobox         # Fedora
sudo apt install distrobox         # Debian 13+ / Ubuntu 24.04+
sudo pacman -S distrobox           # Arch
```

## Usage examples
```sh
# Create a rootless Ubuntu 24.04 container with full host integration
distrobox create --image ubuntu:24.04 --name ubu

# Drop into it — your $HOME, $DISPLAY, GPU, audio are already wired
distrobox enter ubu

# Inside: install whatever the host distro doesn't have
sudo apt update && sudo apt install -y build-essential gdb

# Export a containerized app so it shows up in the host launcher
distrobox enter ubu -- sudo apt install -y inkscape
distrobox-export --app inkscape

# Run a single command in a container without entering interactively
distrobox enter fed -- dnf provides /usr/bin/strace

# List, stop, remove
distrobox list
distrobox stop ubu
distrobox rm ubu --force

# Pin to podman explicitly (default is whatever runtime is installed)
distrobox create --image archlinux:latest --name arch --root=false
```
