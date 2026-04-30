# osv-scanner

A vulnerability scanner that resolves a project's dependencies from
lockfiles, SBOMs, container images, or git commits and matches them
against the open-source OSV.dev database.

## Repo

- URL: https://github.com/google/osv-scanner

## Version

- v2.0.3 (2025 release line)

## License

- Apache-2.0
- License file path in upstream: `LICENSE`

## Install

```sh
go install github.com/google/osv-scanner/v2/cmd/osv-scanner@latest
# or: brew install osv-scanner
```

## Use case category

Software composition analysis (SCA) / supply chain security. Designed
to be runnable both locally by a developer and inside CI as a gating
step.

## Example usage

```sh
# Scan a project's lockfiles recursively, emitting a table to stdout
osv-scanner scan source -r .

# Scan an SBOM produced by another tool and emit SARIF for code review UIs
osv-scanner scan source --sbom=sbom.spdx.json --format=sarif > osv.sarif
```

Expected behavior: prints a per-package table of vulnerable versions
with OSV IDs, fixed versions, and severity. Exits non-zero when at
least one vulnerability is found, which makes it usable as a CI gate
without any extra wrapper script.

## Strengths

- Reads a wide spread of ecosystems out of the box: npm
  (`package-lock.json`, `pnpm-lock.yaml`), Python (`poetry.lock`,
  `requirements.txt`, `uv.lock`), Go (`go.mod`), Rust (`Cargo.lock`),
  Maven, Gradle, Composer, and more.
- Backed by OSV.dev, which aggregates GHSA, RustSec, PyPA,
  Go vulndb, and ecosystem-native advisories — fewer blind spots
  than scanners pinned to a single feed.
- First-class SARIF and JSON output, so results plug straight into
  GitHub code scanning, GitLab security dashboards, or custom
  triage pipelines.

## Limitations

- Surface-level only: it tells you a vulnerable version is present
  but does not do reachability analysis, so noise is real on large
  monorepos.
- Container-image scanning is newer and less battle-tested than
  dedicated image scanners; for deep image scanning most teams still
  pair it with `trivy` or `grype`.

## Comparison

Versus `trivy` (already in this zoo): `trivy` is a broader scanner
covering images, filesystems, IaC, secrets, and licenses; `osv-scanner`
is narrower and more opinionated — purely OSV-backed dependency
scanning — which makes it the cleaner choice when you want a single
authoritative source for OSS CVEs in a polyglot monorepo.

## When to reach for it

- You maintain a polyglot monorepo and want one tool whose findings
  map cleanly back to OSV records (each finding has a permanent
  `OSV-…` or ecosystem-prefixed ID, which makes triage tickets
  durable).
- You want a CI gate that fails the build on any
  high-severity vulnerable dependency, with a small, scriptable
  binary instead of a hosted SaaS scanner.
- You already have an SBOM (SPDX or CycloneDX) and want a fast way
  to ask "are any of these packages currently known-vulnerable?"
  without re-resolving the dependency graph.
