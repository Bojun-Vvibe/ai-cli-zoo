# trivy

- **Repo:** https://github.com/aquasecurity/trivy
- **Version:** v0.70.0 (latest release, 2026-04)
- **License:** Apache-2.0 ([LICENSE](https://github.com/aquasecurity/trivy/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install trivy` · `apt install trivy` (Aqua repo) · `dnf install trivy` · `pacman -S trivy` · prebuilt binaries on the GitHub releases page · Docker `aquasec/trivy`

## What it does

`trivy` is an all-in-one open-source security scanner that
unifies vulnerability scanning, misconfiguration scanning,
secret scanning, license scanning, and SBOM generation behind
a single CLI and a single offline-cacheable database. It scans
container images (`trivy image alpine:3.20`), filesystems and
git repos (`trivy fs .`, `trivy repo https://github.com/...`),
running Kubernetes clusters (`trivy k8s --report summary`),
cloud accounts (`trivy aws --service s3`), and individual IaC
files (Terraform, CloudFormation, Helm, Dockerfile, Kubernetes
manifests). The vulnerability data combines distro advisories
(Alpine secdb, Debian/Ubuntu OVAL, RHEL OVAL, Amazon Linux
ALAS, …) with language-ecosystem advisories (GHSA for npm /
pip / cargo / go / nuget / composer / rubygems / maven, plus
the GitLab Advisory Database) into a single SQLite cache
(`~/.cache/trivy/db/`) that updates incrementally and works
fully offline once primed (`trivy image --download-db-only`).
Output formats include human-readable tables, JSON, SARIF (for
GitHub code-scanning), CycloneDX and SPDX SBOMs, JUnit XML, and
Cosign attestations; `--severity HIGH,CRITICAL --exit-code 1`
makes it a drop-in CI gate. The v0.70.0 release improved
Kubernetes scanning, expanded OSV ecosystem coverage, and
tightened SBOM-as-input flows so you can scan an existing SBOM
(`trivy sbom artifact.cdx.json`) without re-resolving the
dependency graph.

## When to pick it / when not to

Pick `trivy` when you want one binary that covers the full
"shift-left" surface — image, repo, IaC, secrets, SBOM,
clusters — without paying for a SaaS dashboard and without
wiring three separate scanners. The single-binary, single-cache,
offline-capable model is what makes it the default in most CI
pipelines today: a multi-stage Dockerfile build can run `trivy
fs --scanners vuln,secret,misconfig .` in under thirty seconds
on a warm cache, and the SARIF output flows straight into
GitHub's Security tab. The k8s and cloud scanners are the
distinguishing features over single-purpose peers.

Prefer it over [`grype`](https://github.com/anchore/grype) when
you want IaC + secrets + cluster scanning bundled in (grype is
vuln-only but pairs with [`syft`](https://github.com/anchore/syft)
for SBOMs and is faster on pure image scans); over
[`snyk`](https://snyk.io) CLI when you want fully self-hosted
with no account; over `docker scout` when you want vendor-
neutral data sources and IaC coverage. Skip it for runtime
threat detection (use Falco, Tetragon, or
[`tracee`](https://github.com/aquasecurity/tracee) — Trivy is a
static / pre-deploy scanner) and for deep license-compliance
workflows where you need a dedicated SCA platform with policy
inheritance and remediation tracking.

## Example

```bash
# Scan a container image, fail CI on high/critical CVEs
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest

# Scan a repo for vulns, secrets, and IaC misconfigs in one pass, SARIF for GitHub
trivy fs --scanners vuln,secret,misconfig --format sarif -o trivy.sarif .

# Generate a CycloneDX SBOM from a container image
trivy image --format cyclonedx --output myapp.cdx.json myapp:latest
```
