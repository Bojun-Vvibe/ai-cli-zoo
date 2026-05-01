# apko

## What it does
Builds OCI container images directly from APK (Alpine package) declarations — no Dockerfile, no shell layers. Images are reproducible, minimal, and produced from a single declarative YAML manifest.

## Why it's interesting
Pairs with `melange` to produce distroless-style images with deterministic, byte-for-byte reproducible builds and a complete SBOM emitted at build time. Popular foundation for the Chainguard Images / Wolfi distro and a clean alternative to Dockerfile-based builds when supply-chain provenance matters.

## Niche category
Container build / supply-chain — declarative OCI image builder

## Repo
https://github.com/chainguard-dev/apko

## Version pinned
`v1.2.9`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
brew install apko
# or
go install chainguard.dev/apko@latest
```

## Usage examples
```sh
# Build an OCI image tarball from a declarative config
apko build examples/nginx.yaml nginx:latest nginx.tar

# Publish directly to a registry
apko publish examples/nginx.yaml ttl.sh/my-nginx:1h
```
