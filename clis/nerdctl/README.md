# nerdctl

> **Docker-compatible CLI for containerd.** A drop-in
> `docker`-shaped command surface (`run`, `build`, `compose`,
> `pull`, `push`, `images`, `ps`, `exec`, `logs`, `network`,
> `volume`) that talks straight to a `containerd` socket — no
> Docker daemon, with first-class rootless mode, BuildKit,
> CNI networking, and OCI-native features (lazy pulling via
> Stargz/SOCI, image encryption, image signing). Pinned to
> **v2.2.2** ([LICENSE](https://github.com/containerd/nerdctl/blob/v2.2.2/LICENSE),
> Apache-2.0).

Source: <https://github.com/containerd/nerdctl>

## TL;DR

`nerdctl` is what you reach for when you want the `docker` CLI
ergonomics on top of `containerd` directly: no Docker Engine,
no shim daemon, just `containerd` + `runc` + `buildkitd`. The
"full" tarball ships containerd, runc, BuildKit, CNI plugins,
RootlessKit, and the rootless setup script in one archive
(`containerd-rootless-setuptool.sh install`), so a single
`tar xzvf` gets you a working rootless container host. Beyond
parity, it exposes containerd-native features Docker hides:
`nerdctl pull --snapshotter=stargz` for lazy image pulls that
start the container before the layers finish downloading,
`nerdctl image encrypt` / `decrypt` for OCI image encryption,
`nerdctl image sign` / `verify` against a cosign keypair, and
`nerdctl ipfs registry` for IPFS-backed distribution.
`nerdctl compose up` reads an unmodified `compose.yaml`.

## Install

```bash
# Homebrew (macOS — Lima-backed)
brew install nerdctl

# Linux: minimal tarball (nerdctl only, requires containerd already installed)
sudo tar Cxzvf /usr/local/bin \
  https://github.com/containerd/nerdctl/releases/download/v2.2.2/nerdctl-2.2.2-linux-amd64.tar.gz

# Linux: full distribution (nerdctl + containerd + runc + BuildKit + CNI)
sudo tar Cxzvf /usr/local \
  https://github.com/containerd/nerdctl/releases/download/v2.2.2/nerdctl-full-2.2.2-linux-amd64.tar.gz
containerd-rootless-setuptool.sh install
```

## Example

```bash
# Drop-in docker replacement
nerdctl run -d --name web -p 8080:80 nginx:alpine
nerdctl ps
nerdctl logs -f web

# BuildKit-backed multi-arch build, push to a registry
nerdctl build --platform=linux/amd64,linux/arm64 -t ghcr.io/acme/api:1.0 --push .

# Lazy pull with Stargz: container starts before layers finish
nerdctl --snapshotter=stargz run -it --rm \
  ghcr.io/stargz-containers/python:3.10-org bash

# Encrypted image round-trip with cosign-style verification
nerdctl image encrypt --recipient jwe:./pub.pem alpine:3.19 alpine:enc
nerdctl image sign --key cosign.key alpine:enc
nerdctl image verify --key cosign.pub alpine:enc

# Compose without a Docker daemon
nerdctl compose -f compose.yaml up -d
```

## When to use

- You want the `docker` CLI muscle memory but on a
  containerd-only host (Kubernetes nodes, minimal VM images,
  rootless laptops).
- You need OCI-native features Docker does not expose:
  Stargz / SOCI lazy pulls, OCI image encryption, native cosign
  signing/verification, IPFS-backed distribution.
- You want one tarball (`nerdctl-full`) that bootstraps a
  complete rootless container host with no package manager.

## When NOT to use

- You already run Docker Engine and depend on Docker-specific
  surfaces (Swarm mode, Docker Desktop extensions, the Docker
  socket activation contract for sibling containers) — stay on
  `docker`.
- You target macOS / Windows native — nerdctl needs Linux; on
  macOS use [`lima`](../lima/) / [`colima`](../colima/) to host
  it (Homebrew's `brew install nerdctl` does this for you).
- You want a higher-level pod abstraction with systemd-style
  unit generation — pick `podman` instead.
