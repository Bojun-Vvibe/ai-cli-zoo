# kube-score

> **Static analyzer for Kubernetes manifests with reliability +
> security recommendations.** Reads YAML files (or stdin from
> `helm template` / `kustomize build`) and grades each resource
> against ~30 best-practice checks: missing readiness/liveness
> probes, no resource limits, no PodDisruptionBudget, image
> tag `latest`, NetworkPolicy gaps, container running as root.
> Pinned to **v1.20.0**
> (per upstream releases page, verified 2026-05-01)
> ([LICENSE](https://github.com/zegl/kube-score/blob/master/LICENSE),
> MIT).

Source: <https://github.com/zegl/kube-score>

## TL;DR

`kube-score score manifests/*.yaml` parses every Kubernetes
object and runs a fixed catalogue of static checks against it,
then prints findings ranked CRITICAL / WARNING / OK with a
short rationale and a fix hint per finding. Output formats
include human (default), JSON, ndjson, sarif, and CI (one line
per finding for grep-friendly pipelines). Severity floor is
configurable (`--exit-one-on-warning`) so a CI gate fails the
PR when any new manifest drops below the team's bar; per-object
opt-outs live in YAML annotations
(`kube-score/ignore: pod-networkpolicy`) so legitimate
exceptions are reviewable in the diff instead of buried in CI
config.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kube-score

# Go install
go install github.com/zegl/kube-score/cmd/kube-score@latest

# Docker
docker run --rm -v "$PWD":/work \
  zegl/kube-score:v1.20.0 score /work/manifests/*.yaml
```

## Example

```bash
# Score a directory of raw manifests
kube-score score manifests/*.yaml

# Pipe a rendered Helm chart in and fail CI on any warning
helm template my-release ./chart \
  | kube-score score --output-format ci --exit-one-on-warning -

# SARIF for GitHub code-scanning
kube-score score --output-format sarif manifests/*.yaml \
  > kube-score.sarif

# Skip a noisy check globally
kube-score score \
  --ignore-test container-image-pull-policy \
  --ignore-test pod-networkpolicy \
  manifests/*.yaml
```

## When to use

- You want a fast, opinionated PR gate that catches the boring
  reliability mistakes (no probes, no limits, no PDB) before
  the manifest ever hits a cluster.
- Your manifests come from `helm template` or `kustomize build`
  and you want one tool that scores the rendered YAML without
  needing a live cluster.
- You want SARIF output that drops straight into GitHub
  code-scanning without writing a converter.

## When NOT to use

- You want *schema validation* (does this `apiVersion` / field
  even exist?) — that is `kubeconform` or `kubectl --dry-run`,
  not `kube-score`.
- You want *policy* with custom rules per project (e.g. "only
  these registries allowed", "must carry these labels") — that
  is `kyverno` / `polaris` / `conftest`, which let you author
  rules. `kube-score`'s rule set is fixed.
- You want a security posture scan mapped to CIS / NSA controls
  — reach for `kubescape` or `trivy config`. `kube-score`
  overlaps but is reliability-first, not compliance-first.
