# popeye

> **Read-only Kubernetes cluster sanitizer.** Scans a live
> cluster (or a kubeconfig context) and reports misconfigured
> or unhealthy resources — dangling services, container images
> without tags, missing resource requests/limits, unused
> ConfigMaps / Secrets, RBAC sprawl, expired certs — with a
> grade per namespace and per resource kind. Pinned to **v0.21.5**
> (per upstream releases page, verified 2026-05-01)
> ([LICENSE](https://github.com/derailed/popeye/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/derailed/popeye>

## TL;DR

`popeye` walks every API resource the current kubeconfig can
list and runs a fixed set of "sanitizers" against each kind
(Pods, Deployments, Services, Ingresses, RBAC, Nodes, PVs/PVCs,
HPAs, …). The output is a per-namespace report card with
severity-coded findings (✅ ok, 🔵 info, 🟡 warn, 🔴 error) plus
a numeric grade A–F so you can track drift over time. Output is
human (default), JSON, YAML, JUnit, or HTML — and a `--save`
mode plus optional S3 / GCS / Azure-Blob push makes it usable
as a recurring cron job that drops a report into a bucket per
cluster, per day.

## Install

```bash
# Homebrew (macOS / Linux)
brew install derailed/popeye/popeye

# Go install
go install github.com/derailed/popeye@latest

# Docker
docker run --rm \
  -v "$HOME/.kube/config":/root/.kube/config \
  derailed/popeye:v0.21.5 -A
```

## Example

```bash
# Scan every namespace, human-readable output
popeye -A

# Single namespace, JSON for downstream tooling
popeye -n payments -o json --save

# CI gate: fail (non-zero exit) if any resource scores below B
popeye -A --min-score 80 -o junit > popeye.xml
```

## When to use

- You inherited a cluster and want a one-page diagnostic of
  what is misconfigured *right now* — no install, no agent,
  just a kubeconfig.
- You want a recurring "cluster report card" artifact (HTML /
  JSON dropped to a bucket) so drift is visible week-over-week.
- You're prepping for an upgrade and want to flush out
  deprecated apiVersions, dangling Services, and unused
  ConfigMaps before they bite.

## When NOT to use

- You want *policy enforcement* (block bad manifests at admit
  time) — that is `kyverno` / `gatekeeper`, not a scanner.
- You want CVE / SBOM scanning of container images — reach for
  `trivy` or `grype`.
- You only have static manifests (no live cluster); `popeye`
  needs an API server to talk to. Use `kube-linter` /
  `kubeconform` for static analysis.
