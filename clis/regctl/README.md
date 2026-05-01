# regctl

## What it does
A daemonless OCI registry client from the `regclient` project. `regctl image copy src ref dst ref` mirrors images between registries without pulling layers to local disk; `regctl image inspect`, `regctl manifest get`, `regctl tag ls`, and `regctl image mod` give scriptable access to manifests, indexes, layers, and tag history across Docker Hub, GHCR, ECR, GCR, Harbor, Quay, and any OCI-compliant registry. Includes `regbot` (policy-driven mirroring) and `regsync` (declarative registry sync) as sibling binaries.

## Why it's interesting
Where `crane` is Google's image-surgery toolkit and `skopeo` is the RPM-world incumbent, `regctl` covers the same ground with first-class support for OCI 1.1 referrers (the mechanism Sigstore/Cosign uses to attach signatures and attestations to images), digest-pinned multi-arch index manipulation, and a clean `image mod` subcommand for non-destructive edits (re-tagging timestamps, stripping layers, replacing entrypoints) that emit a new digest without rebuilding. The companion `regsync` reads a YAML policy and continuously mirrors a curated set of upstream images to your internal registry — useful for air-gapped clusters and for pinning third-party dependencies against upstream-tag rewrites.

## Niche category
OCI registry client — daemonless, referrer-aware, mirror-capable

## Repo
https://github.com/regclient/regclient

## Version pinned
`v0.11.3`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
# macOS / Linux via Homebrew
brew install regclient

# Or download the static binary from releases
curl -L https://github.com/regclient/regclient/releases/latest/download/regctl-linux-amd64 \
  -o /usr/local/bin/regctl && chmod +x /usr/local/bin/regctl
```

## Usage examples
```sh
# Copy an image between two registries without local pull
regctl image copy ghcr.io/me/app:v1 quay.io/me/app:v1

# Inspect a multi-arch manifest list and resolve a digest
regctl manifest get ghcr.io/me/app:v1 --format '{{json .}}'

# List all referrers (signatures, SBOMs, attestations) for a digest
regctl artifact list ghcr.io/me/app@sha256:abc...

# Non-destructive image mutation — replace the entrypoint and re-tag
regctl image mod ghcr.io/me/app:v1 --entrypoint '["/app","--prod"]' \
  --create ghcr.io/me/app:v1-prod
```

## When to pick `regctl` vs alternatives
- **vs [`crane`](../crane/)**: both are excellent daemonless registry tools. Pick `regctl` when you need OCI 1.1 referrers, `regsync`-style declarative mirroring, or `image mod` for post-hoc edits. Pick `crane` when you're already deep in the `go-containerregistry` ecosystem (`ko`, `kontain.me`, `gcrane`).
- **vs `skopeo`**: `skopeo` is the RPM/Podman-world standard and ships in every Red Hat distro; `regctl` is a single static binary with arguably cleaner UX and better OCI 1.1 support. On RHEL hosts where `skopeo` is already installed, prefer `skopeo`; in greenfield CI, `regctl` is lighter.
- **vs [`oras`](../oras/)**: `oras` is purpose-built for pushing/pulling *arbitrary* OCI artifacts (Helm charts, WASM modules, ML models). `regctl` is image-first with referrer support; use `oras` when the artifact isn't an image at all.
- **vs `docker pull && docker push`**: `regctl image copy` never materializes layers locally — orders of magnitude faster for cross-region mirroring of large images, and works without a daemon.
