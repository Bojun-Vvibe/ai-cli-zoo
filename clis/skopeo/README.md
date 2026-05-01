# skopeo

> **Work with remote container image registries without a daemon
> and without pulling the image** — a single Go binary from the
> containers project that inspects, copies, signs, deletes, and
> syncs OCI / Docker v2 images directly between any two
> transports (registry, local OCI layout, `containers-storage`,
> Docker daemon, tarball, dir). Pinned to **v1.20.0**
> (released 2025-09-29,
> [LICENSE](https://github.com/containers/skopeo/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/containers/skopeo>

## TL;DR

`skopeo` is the answer when you need to do *registry plumbing* —
mirror an image from Docker Hub to a private registry, peek at a
manifest before pulling 3 GiB of layers, copy across architectures,
verify a cosign signature, or wire image promotion into CI — and
you do not want to pay the cost of a running container engine to do
it. No daemon, no root, no `docker pull && docker tag && docker
push` round-trip through local storage. The same binary speaks
every common transport (`docker://`, `oci:`, `dir:`,
`containers-storage:`, `docker-daemon:`, `docker-archive:`) so a
copy is one verb between two URIs and the bytes stream registry to
registry without ever touching local disk.

## Install

```bash
# Homebrew (macOS / Linux)
brew install skopeo

# Linux package managers
# Debian / Ubuntu: apt install skopeo
# Fedora / RHEL: dnf install skopeo
# Arch: pacman -S skopeo
# Alpine: apk add skopeo
# Nix: nix-env -iA nixpkgs.skopeo

# Static binary (any OS)
# https://github.com/containers/skopeo/releases

# verify
skopeo --version    # skopeo version 1.20.0
```

## Examples

```bash
# inspect a remote image without pulling layers (manifest + config + labels)
skopeo inspect docker://docker.io/library/alpine:3.20
skopeo inspect --raw docker://ghcr.io/owner/app:latest | jq .

# list every tag a repo exposes
skopeo list-tags docker://docker.io/library/postgres

# mirror an image registry-to-registry, no local pull
skopeo copy \
  docker://docker.io/library/redis:7-alpine \
  docker://registry.internal.example.com/cache/redis:7-alpine

# copy a multi-arch manifest list (all platforms preserved)
skopeo copy --multi-arch all \
  docker://docker.io/library/nginx:1.27 \
  oci:./nginx-mirror:1.27

# sync a whole repo or a yaml-listed set of images on a schedule
skopeo sync --src docker --dest dir \
  docker.io/library/alpine ./offline-mirror/

# delete a tag from a registry that allows it
skopeo delete docker://registry.internal/old-app:abandoned
```

## Use when

- You are building a **CI image-promotion pipeline** (build →
  staging registry → smoke tests → promote to prod registry) and
  every step is "copy this digest from A to B" — `skopeo copy` is
  one process, no daemon, no scratch disk needed.
- You run an **air-gapped mirror** and need to pull a curated set
  of upstream images into a local registry over a periodic sync
  job — `skopeo sync --src yaml` reads a list, copies in
  parallel, idempotent on re-runs (skips by digest).
- You want to **inspect an image's labels / entrypoint / size /
  layers / signatures before pulling it** — `skopeo inspect`
  fetches only the manifest + config blob, not the layers.
- You need the same binary to talk to a **local Docker daemon, a
  rootless `containers-storage` (podman) database, an OCI layout
  on disk, a tarball, and a remote registry** with one URI scheme
  per transport.
- Pair with [`cosign`](../cosign/) (sign / verify the image
  skopeo just copied), [`crane`](../crane/) (Google's overlapping
  registry CLI — crane is leaner, skopeo's transport matrix is
  wider), [`oras`](../oras/) (push non-image OCI artifacts to the
  same registry), [`regctl`](../regctl/) (yet another registry
  CLI; use whichever your team already standardised on).

Skip `skopeo` when a `docker pull` / `docker push` cycle is fine
and the daemon is already running — the win is *not needing a
daemon*. For anything beyond image plumbing (build, run, network,
volumes) reach for [`podman`](../podman/) / [`nerdctl`](../nerdctl/)
/ Docker; skopeo deliberately stops at the registry boundary.
