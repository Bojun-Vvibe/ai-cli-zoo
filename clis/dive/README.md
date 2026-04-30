# dive

> **Interactive container image explorer.** Opens a TUI that
> lays out every layer of a Docker / OCI image side-by-side with
> its filesystem tree, marking files added / modified / removed
> per layer and computing an "image efficiency" score that
> quantifies wasted space from files copied-then-deleted across
> layers. Pinned to **v0.13.1**
> ([LICENSE](https://github.com/wagoodman/dive/blob/main/LICENSE),
> MIT).

Source: <https://github.com/wagoodman/dive>

## TL;DR

`dive <image>` mounts each layer in a read-only view and lets
you walk the diff with arrow keys: left pane is the layer list
with `Cmd`, `Size`, and cumulative wasted bytes; right pane is
the merged filesystem at that layer with per-file colour codes
(green added, yellow modified, red removed). The same engine
runs headless under `CI=true dive --ci <image>`, which fails
the build when efficiency drops below `--lowestEfficiency` or
wasted bytes exceed `--highestWastedBytes` / a percent
threshold — turning "is my Dockerfile leaking 200 MB of
build-time apt cache?" into a CI gate.

## Install

```bash
# Homebrew (macOS / Linux)
brew install dive

# Docker (no install; mount the host docker socket)
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:v0.13.1 <image>

# Go install
go install github.com/wagoodman/dive@latest
```

## Example

```bash
# Interactive walk of a local image
dive my-app:latest

# CI gate: fail if image efficiency < 95 % or > 50 MB wasted
CI=true dive --ci \
  --lowestEfficiency 0.95 \
  --highestWastedBytes 50000000 \
  my-app:latest

# Inspect an image straight from a registry-pulled tarball
docker save alpine:3.19 -o alpine.tar
dive docker-archive://alpine.tar
```

## When to use

- You suspect a layer is bloated (build cache, `.git`, leftover
  apt lists) and want to see *which* files are the cost before
  rewriting the Dockerfile.
- You want a hard CI check on image efficiency / wasted bytes
  alongside the usual lint and vuln scans.
- You inherited an image and need to understand what each
  `RUN` step actually changed on disk.

## When NOT to use

- You want a vuln scan or SBOM — that is `trivy` / `grype` /
  `syft`, not `dive`.
- The image is huge and you only need a one-shot size summary;
  `docker history` is faster for a flat layer table.
- You need to inspect a *running* container's filesystem;
  `dive` analyses the image, not the live container.
