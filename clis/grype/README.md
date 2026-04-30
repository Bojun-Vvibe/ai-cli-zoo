# grype

> **Vulnerability scanner for container images and filesystems.**
> Reads a Syft SBOM (or generates one inline) and matches its
> packages against a curated vulnerability database aggregated
> from NVD, GitHub Security Advisories, and ~15 distro feeds.
> Pinned to **v0.111.1**
> ([LICENSE](https://github.com/anchore/grype/blob/v0.111.1/LICENSE),
> Apache-2.0).

Source: <https://github.com/anchore/grype>

## TL;DR

`grype` is the scanner half of Anchore's open-source supply-
chain pair: [`syft`](../syft/) catalogs *what is in* an image
or directory, `grype` matches that catalog against a CVE feed.
Targets are URI-shaped: `grype alpine:3.20` (pull and scan a
registry image), `grype dir:/src` (scan a working tree),
`grype sbom:./sbom.spdx.json` (scan a pre-generated SBOM —
the production pattern: build SBOM once at image-build time,
re-scan it nightly without re-pulling the image), `grype
docker-archive:./img.tar` (scan a saved tar), `grype
oci-dir:./img/` (scan an exploded OCI bundle). The vulnerability
database is built nightly by Anchore, distributed as a single
SQLite file via `grype db update` (auto-runs by default,
disable with `--db-auto-update=false` for air-gapped CI),
and contains entries from NVD, GHSA, the Alpine secdb, the
Amazon Linux ALAS feed, the Debian Security Tracker, the RHEL
OVAL feed, the Ubuntu USN feed, Wolfi, Chainguard, and several
language-ecosystem advisory databases. Output shapes include
`table` (default), `json`, `cyclonedx-json`, `sarif` (drop
into GitHub code-scanning), and `template` (Go-template for
custom report rows). `--fail-on high` makes it a CI gate;
`.grype.yaml` `ignore:` rules suppress false positives by
CVE / package / fix-state with an audit-friendly diff.

## Install

```bash
# Homebrew (macOS / Linux)
brew install grype

# Official installer (pins a version)
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh \
  | sh -s -- -b /usr/local/bin v0.111.1

# Go install
go install github.com/anchore/grype@v0.111.1
```

## Example

```bash
# One-shot scan of a registry image, table output
grype alpine:3.20

# Generate an SBOM with syft, then scan that SBOM repeatedly
syft alpine:3.20 -o spdx-json > alpine.sbom.spdx.json
grype sbom:./alpine.sbom.spdx.json

# CI gate: fail the job on any High or Critical CVE with a fix
grype dir:. --fail-on high --only-fixed

# Emit SARIF for GitHub code-scanning upload
grype ghcr.io/acme/api:1.4.2 -o sarif > grype.sarif

# Suppress a known false-positive across the repo
cat > .grype.yaml <<'YAML'
ignore:
  - vulnerability: CVE-2024-12345
    package:
      name: libfoo
    reason: "upstream confirmed not exploitable in our config"
YAML
```

## When to use

- You ship container images and want a fast, scriptable CVE gate
  in the build pipeline without standing up a server-side scanner.
- You already build SBOMs with [`syft`](../syft/) at image-build
  time and want to re-scan the same SBOM nightly as new CVEs are
  published, without re-pulling the image.
- You want SARIF output that GitHub Advanced Security / Defender
  for DevOps consume natively, plus per-CVE ignore rules that
  live in the repo as reviewable YAML.

## When NOT to use

- You need verified-only "is this credential actually live"
  scanning of a repo's secrets — that is
  [`trufflehog`](../trufflehog/)'s job, not grype's.
- You want runtime threat detection (process / syscall / network
  anomalies inside a running container) — pick Falco, Tetragon,
  or [`kubescape`](../kubescape/) for posture; grype is a
  *static* pre-deploy scanner.
- Your supply-chain question is "did this commit / image come
  from where it claims" — that is [`cosign`](../cosign/) +
  Sigstore signature verification, not vulnerability matching.
