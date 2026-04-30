# syft

> **SBOM generator for container images and filesystems.** Scans
> a target (Docker image, OCI archive, directory, tarball, single
> binary) and emits a software bill of materials in CycloneDX,
> SPDX, or its own native JSON — covering 30+ ecosystems
> (apk/dpkg/rpm OS packages, npm, pip, gem, cargo, go modules,
> maven, nuget, swift, dart, conan, hex, etc.) plus binary
> classifier matches for unmanaged binaries. Pinned to **v1.14.2**
> ([LICENSE](https://github.com/anchore/syft/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/anchore/syft>

## TL;DR

`syft <target>` produces a complete inventory of "what is inside"
without executing the artifact. Drives downstream supply-chain
work: hand the SBOM to `grype` for vuln scan, to `cosign attest`
for signed attestations, or to compliance pipelines that demand
SPDX. Reads images directly from a registry (`syft
registry:alpine:3.19`), local Docker, podman, containerd, or a
plain directory (`syft dir:./repo`). Output formats are flag-
selectable — `-o cyclonedx-json`, `-o spdx-json`, `-o table`,
`-o syft-json` — and multiple `-o fmt=path` writes parallel
formats in one pass.

## Install

```bash
# Homebrew (macOS / Linux)
brew install syft

# Install script (any platform)
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh \
  | sh -s -- -b /usr/local/bin

# Go install
go install github.com/anchore/syft/cmd/syft@latest

# Docker
docker run --rm anchore/syft:v1.14.2 alpine:3.19
```

## Example

```bash
# SBOM of a remote image, CycloneDX JSON, written to file
syft registry:alpine:3.19 -o cyclonedx-json=alpine.cdx.json

# SBOM of a local source tree, table view for humans
syft dir:./myrepo -o table

# Pipe straight into grype for a vuln scan with one parse
syft my-image:latest -o syft-json | grype
```

## When to use

- You need a CycloneDX or SPDX SBOM as a release artifact and do
  not want to hand-write a generator per ecosystem.
- Your vuln scanner or attestation tool wants an SBOM as input
  rather than re-scanning the image itself.
- You are auditing an image you did not build and need to know
  every OS package and language dependency baked in.

## When NOT to use

- You only want a vuln verdict — `grype` / `trivy` can read the
  image directly.
- You need runtime/behavioral analysis — syft is purely static
  metadata extraction.
- The artifact is a fully unmanaged binary with no detectable
  language signatures; the binary classifier covers common Go /
  Rust / Python interpreters but not arbitrary C executables.
