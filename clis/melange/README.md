# melange

## What it does
A declarative apk package builder for the Wolfi / Alpine ecosystem. You write a `melange.yaml` describing a package (name, version, dependencies, build pipeline), and `melange build` produces a signed `.apk` plus an in-toto SLSA provenance attestation, runnable inside a sandbox VM (`--runner qemu`), bubblewrap, or a Docker / podman / containerd container. Pairs natively with [`apko`](../apko/): melange builds the apks, apko assembles them into a distroless OCI image with a generated SBOM. The same yaml runs locally on a laptop and inside CI without modification.

## Why it's interesting
The "container base image" problem has historically been: pull `ubuntu:22.04` (700 MB, 200+ CVEs at any moment, no SBOM), or build distroless by hand and lose a week. melange + apko is Chainguard's answer: every binary in the resulting image came from an apk you can rebuild from source, every apk is signed, and every image ships an SBOM and a SLSA provenance. The pipeline DSL is small and copy-pasteable (`fetch` + `patch` + `autoconf/configure` + `make/install` + `strip`), the sandbox is hermetic (no host network unless you opt in), and multi-arch (`--arch amd64,arm64,riscv64`) is one flag rather than a build matrix. For supply-chain-conscious teams who outgrew "FROM alpine" but don't want to maintain a Yocto / Buildroot / NixOS path, melange is the smallest declarative on-ramp; for the wider ecosystem, it's the build tool behind the Wolfi distro and Chainguard's distroless image catalogue, so the yaml schema is battle-tested at thousands-of-packages scale.

## Niche category
Declarative apk package builder with hermetic sandbox + SBOM / SLSA provenance — the upstream half of the apko-based distroless image story

## Repo
https://github.com/chainguard-dev/melange

## Version pinned
`v0.50.4`

## License
- SPDX: `Apache-2.0`
- License file in upstream repo: `LICENSE`

## Install
```sh
# Homebrew (macOS / Linux)
brew install melange

# Go install (requires Go 1.22+)
go install chainguard.dev/melange@v0.50.4

# Pre-built binary
curl -L https://github.com/chainguard-dev/melange/releases/download/v0.50.4/melange_0.50.4_linux_amd64.tar.gz | tar xz
sudo install melange /usr/local/bin/

# Or run as a container (no host install)
docker run --rm -v "$PWD:/work" -w /work cgr.dev/chainguard/melange:latest version
```

## Usage
```yaml
# hello.yaml — minimal package
package:
  name: hello
  version: 2.12.1
  epoch: 0
  description: GNU hello, packaged with melange
  copyright:
    - license: GPL-3.0-or-later

environment:
  contents:
    repositories:
      - https://packages.wolfi.dev/os
    keyring:
      - https://packages.wolfi.dev/os/wolfi-signing.rsa.pub
    packages:
      - build-base
      - busybox
      - ca-certificates-bundle

pipeline:
  - uses: fetch
    with:
      uri: https://ftp.gnu.org/gnu/hello/hello-${{package.version}}.tar.gz
      expected-sha256: 8d99142afd6475a8e2d7cf74b0cdfdc4c755c6f120f93ac80fb8d61c20eb1338
  - uses: autoconf/configure
  - uses: autoconf/make
  - uses: autoconf/make-install
  - uses: strip
```

```sh
# Build the apk in a hermetic sandbox (default runner is docker; bwrap / qemu / lima also supported)
melange keygen   # one-time signing key
melange build hello.yaml --arch amd64,arm64 --signing-key melange.rsa

# Inspect the resulting apk + SBOM
ls packages/x86_64/      # hello-2.12.1-r0.apk + .sig + .spdx.json

# Hand off to apko to assemble a distroless image
apko build apko.yaml hello:latest hello.tar --arch amd64,arm64

# CI-friendly variant — print SBOM, fail if pipeline diverges from lockfile
melange build hello.yaml --runner docker --generate-index --sbom-path sbom/

# Bump a recipe automatically (chainguard-managed packages)
melange bump hello.yaml 2.12.2
```

## When to pick `melange` vs alternatives
- **Pairs with [`apko`](../apko/)**: melange produces apks; apko assembles apks into distroless OCI images. They are designed as a two-step pipeline. If you only need to *consume* a Chainguard image, you don't need either; if you need to add a custom binary to a distroless base, you need both.
- **vs writing a `Dockerfile FROM alpine`**: Dockerfile gives you imperative shell with no provenance, no SBOM, no signature, and a layer cache that drifts. melange gives you a hermetic sandbox + signed apk + SBOM + reproducible builds at the cost of learning one yaml schema.
- **vs `nix build`**: Nix has stronger reproducibility guarantees and a much larger ecosystem but a much steeper learning curve. Pick melange when your team already speaks Alpine/apk and wants 90% of the supply-chain wins for 10% of the Nix tax; pick Nix when you need NixOS-grade hermeticism across the entire system.
- **vs `dpkg-buildpackage` / `rpmbuild` / Gentoo ebuilds**: those target full-fat distros with init systems and shell environments. melange targets the distroless / scratch end of the spectrum where the resulting image has no shell at all.
- **vs Buildpacks / `ko`**: [`ko`](../ko/) is for Go single-binary apps and skips package management entirely. Buildpacks auto-detect language and bundle a runtime. melange is the right pick when you need a *system package* (a C/Rust/Python/Java runtime, an Nginx-with-modules build, a CUDA component) consumed by other apks or by an apko-built image, not just a single app binary.
