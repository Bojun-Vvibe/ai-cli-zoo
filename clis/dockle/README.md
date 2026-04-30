# dockle

> **Container image linter for build-time best practices and
> CIS Docker Benchmark checks.** Scans an OCI / Docker image
> (local daemon, tarball, or registry) and reports violations
> like running as root, missing `HEALTHCHECK`, sensitive files
> committed into a layer (`.npmrc`, `.aws/credentials`,
> `id_rsa`), apt-cache leftovers, and the full CIS Docker
> Benchmark image-section. Pinned to **v0.4.15**
> (per upstream releases page, verified 2026-05-01)
> ([LICENSE](https://github.com/goodwithtech/dockle/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/goodwithtech/dockle>

## TL;DR

`dockle <image>` runs a fixed checklist (CIS Docker Benchmark
4.x image checks + a curated "best practices" set) against the
image's config blob and every layer's filesystem, then prints
a severity-coded list (FATAL / WARN / INFO / SKIP / PASS) with
a one-line remediation for each finding. Each check has a
stable code (`CIS-DI-0001`, `DKL-DI-0006`, …) so you can
suppress known-accepted findings via `--accept-key` /
`-DOCKLE_IGNORES`, and `--exit-code 1 --exit-level fatal` makes
it a CI gate that complements (does not replace) a vuln scanner.

## Install

```bash
# Homebrew (macOS / Linux)
brew install goodwithtech/r/dockle

# Docker (no install)
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  goodwithtech/dockle:v0.4.15 my-app:latest

# Go install
go install github.com/goodwithtech/dockle/cmd/dockle@latest
```

## Example

```bash
# Lint a local image, fail the build on any FATAL finding
dockle --exit-code 1 --exit-level fatal my-app:latest

# Scan a registry image directly (no docker pull needed)
dockle ghcr.io/example/api:1.4.2

# Suppress a known-accepted finding, JSON for CI ingest
DOCKLE_IGNORES=CIS-DI-0001 dockle -f json -o dockle.json my-app:latest
```

## When to use

- You want a CI check that catches "image runs as root", "no
  `HEALTHCHECK`", and "an SSH key got copied into a layer"
  *before* the image ships.
- You need a CIS Docker Benchmark report for compliance with
  stable check codes you can suppress and audit.
- You already run `trivy` / `grype` for CVEs and want a
  separate, complementary signal on image *hygiene*, not
  package vulnerabilities.

## When NOT to use

- You want CVE scanning of OS packages and language deps —
  that is `trivy` / `grype`, not `dockle`.
- You want to *explore* layer contents interactively — that is
  `dive`, which visualises wasted bytes per layer.
- You want runtime / cluster posture checks — reach for
  `kubescape`, `popeye`, or `kyverno`.
