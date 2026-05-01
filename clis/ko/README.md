# ko

## What it does
A container image builder for Go applications that produces minimal OCI images **without a Dockerfile and without a Docker daemon**. `ko build ./cmd/server` cross-compiles the Go binary, layers it on a distroless base, and pushes the resulting image to a registry — all from a single Go-aware command. `ko apply -f config/` rewrites Kubernetes manifests on the fly, replacing `image: ko://github.com/me/app/cmd/server` references with the freshly-pushed digest before piping to `kubectl apply`.

## Why it's interesting
Removes the entire Dockerfile authoring layer for Go services: no base-image pinning, no multi-stage builds, no `COPY --from=builder`, no daemon socket. Because builds are reproducible Go cross-compiles (not layered shell commands), images are byte-identical across machines and CI, and SBOMs are generated automatically. Pairs naturally with `cosign` (auto-signing on push), `crane`/`regctl` (post-hoc image surgery), and GitOps flows where the manifest-rewrite step replaces the usual "build → tag → sed → commit" dance with one command.

## Niche category
Containerless container builder — Go-native, daemonless, distroless

## Repo
https://github.com/ko-build/ko

## Version pinned
`v0.18.1`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
# macOS / Linux via Homebrew
brew install ko

# Or via go install
go install github.com/google/ko@latest
```

## Usage examples
```sh
# Build and push an image for a Go main package
KO_DOCKER_REPO=ghcr.io/me ko build ./cmd/server

# Rewrite ko:// references in manifests, push images, then apply
KO_DOCKER_REPO=ghcr.io/me ko apply -f config/

# Local-only build into the docker daemon for testing
ko build --local ./cmd/server

# Resolve manifests without applying (CI / GitOps commit step)
ko resolve -f config/ > rendered.yaml
```

## When to pick `ko` vs alternatives
- **vs Dockerfile + `docker build`**: pick `ko` when the project is pure Go and you want reproducible, daemonless, distroless images for free; pick a Dockerfile when you need non-Go runtime deps, CGO with system libraries, or fine-grained layer control.
- **vs [`apko`](../apko/)**: `apko` builds images from declarative APK package lists (any language, any binary); `ko` is Go-source-aware and rebuilds on every code change. Use `apko` for base images and polyglot services, `ko` for Go application images.
- **vs [`crane`](../crane/)**: `crane` manipulates existing images (copy, mutate, append layers); `ko` produces them from source. They compose — build with `ko`, retag/copy across registries with `crane`.
- **vs Buildpacks (`pack`)**: Buildpacks are polyglot and opinionated about the full app lifecycle; `ko` is Go-only and does exactly one thing — turn `main.go` into a pushed image — with zero configuration surface.
