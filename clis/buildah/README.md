# buildah

- **Repo:** https://github.com/containers/buildah
- **Version:** v1.43.1
- **License:** Apache-2.0 — [LICENSE](https://github.com/containers/buildah/blob/main/LICENSE)
- **Category:** OCI image builder / daemonless container tooling

## What it is

`buildah` is a daemonless CLI for building OCI / Docker container
images, maintained as part of the `containers` project (alongside
Podman, Skopeo, and CRI-O). Unlike `docker build`, it does not
require a long-running root daemon: every command operates directly
on a *working container* (a writable scratch container backed by a
storage driver) and on local OCI images, using the same
`containers/storage` and `containers/image` libraries that Podman uses.

## Why it's interesting

- **Build without a Dockerfile, or with one — your choice.** `buildah
  bud` consumes a Dockerfile/Containerfile, but `buildah from` +
  `buildah run` + `buildah copy` + `buildah commit` lets you script
  image construction in plain shell with full control over each
  layer, mount, and commit message. Great for from-scratch images.
- **Rootless and daemonless.** With user-namespace mappings
  (`/etc/subuid`, `/etc/subgid`) you can build production images as
  an unprivileged user — no `dockerd`, no `/var/run/docker.sock`
  exposure, no privilege-escalation footgun in CI runners.
- **Mount the working container's filesystem.** `buildah mount $ctr`
  exposes the container rootfs as a host path, so you can use
  *host* tools (rpm, dnf, apt, restorecon, custom installers) to
  populate it — handy for hermetic, reproducible image bases that a
  `RUN` line in a Dockerfile cannot express cleanly.
- **First-class multi-arch.** `buildah manifest create` /
  `buildah manifest add` build OCI image indexes (manifest lists)
  natively, including with `--platform linux/arm64,linux/amd64`
  cross-build via QEMU.
- **Shares storage with Podman.** Images built with `buildah` are
  immediately runnable with `podman run`; no daemon socket, no
  separate registry round-trip needed for local iteration.

## Install

```bash
# Fedora / RHEL
sudo dnf install -y buildah

# Debian / Ubuntu
sudo apt-get install -y buildah

# macOS (via podman machine; buildah ships in the VM)
brew install podman && podman machine init && podman machine ssh -- buildah version

# Verify
buildah version
```

## Quick start

```bash
ctr=$(buildah from alpine:3.20)
buildah run $ctr -- apk add --no-cache curl
buildah config --entrypoint '["/usr/bin/curl"]' $ctr
buildah commit $ctr my-curl:latest
buildah push my-curl:latest docker://registry.example.com/my-curl:latest
```
