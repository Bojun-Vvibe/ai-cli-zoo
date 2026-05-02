# pack

- **Repo:** https://github.com/buildpacks/pack
- **Version:** v0.40.3
- **License:** Apache-2.0 — [LICENSE](https://github.com/buildpacks/pack/blob/main/LICENSE)
- **Category:** Container image build / Cloud Native Buildpacks (CNB)

## What it is

`pack` is the reference CLI for the [Cloud Native Buildpacks](https://buildpacks.io)
project (a CNCF graduated project). It turns application source code
into runnable OCI container images **without a Dockerfile**, by
composing a *builder* image (an ordered set of buildpacks plus a
*stack* base image) against your source tree. Each buildpack detects
whether it applies (Node? Go? JVM? Python?) and contributes a layer
with its dependencies; the result is a deterministic, rebase-able,
SBOM-emitting OCI image.

## Why it's interesting

- **Rebase, not rebuild.** Because the OS layer is a separate
  *run image*, `pack rebase my-app:latest` swaps in a patched base
  image (e.g., for a CVE) without re-running buildpacks or
  re-uploading app layers. CI cost and registry bandwidth drop
  dramatically for large fleets.
- **Reproducible by construction.** Layers are content-addressed and
  built with fixed timestamps; the same source + same builder
  produces byte-identical images, which makes supply-chain attestation
  (cosign/SLSA) tractable.
- **SBOM out of the box.** `pack build --sbom-output-dir` emits
  CycloneDX / SPDX / Syft JSON per layer, so you know exactly which
  buildpack contributed which dependency.
- **No Dockerfile, no root.** Buildpacks run as an unprivileged user
  in a sandboxed builder, which removes a large class of "curl |
  bash as root in a Dockerfile" footguns common to handwritten
  container builds.
- **Pluggable builders.** Heroku, Paketo, Google Cloud, and others
  publish maintained builders; `pack builder suggest` lists them and
  `pack builder inspect` shows the exact buildpack order, stack, and
  lifecycle version pinned inside.

## Install

```bash
# macOS (Homebrew)
brew install buildpacks/tap/pack

# Linux (binary)
curl -sSL "https://github.com/buildpacks/pack/releases/download/v0.40.3/pack-v0.40.3-linux.tgz" \
  | tar -C /usr/local/bin/ --no-same-owner -xzv pack

# Verify
pack version
```

## Quick start

```bash
pack builder suggest
pack build my-app --builder paketobuildpacks/builder-jammy-base
docker run --rm -p 8080:8080 my-app
```
