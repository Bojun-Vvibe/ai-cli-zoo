# kube-linter

> **Static analysis for Kubernetes YAML and Helm charts that flags
> production-readiness and security misconfigurations *before* `kubectl
> apply` ever runs.** Walks any directory of manifests (or a rendered
> Helm chart via `--chart`) against ~50 built-in checks — missing
> `livenessProbe`/`readinessProbe`, `runAsNonRoot: false`, `privileged:
> true`, `hostNetwork`, missing CPU/memory requests, latest-tag images,
> default `ServiceAccount` mounted into a pod, dangling `NetworkPolicy`
> selectors, deprecated API versions — and emits SARIF / JSON / plain
> text suitable for CI gating. Niche tag: **devops / kubernetes
> security**. Pinned to **v0.8.3**
> ([LICENSE](https://github.com/stackrox/kube-linter/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/stackrox/kube-linter>

## TL;DR

`kube-linter lint ./manifests` walks every `*.yaml` under the path,
parses each document as a typed Kubernetes object, and runs the
default check set against it — finishing in a couple of seconds even
for hundreds of manifests because there is no API server round-trip
and no admission webhook. Custom checks are pure YAML
(`templates: [run-as-non-root]` matched against `objectKinds: [Pod,
Deployment, StatefulSet]` with selectors), so a platform team can
ship an org-wide policy file without writing Go or Rego. Pairs
naturally with `helm template ... | kube-linter lint -` for chart
authors who want to lint the *rendered* output, not the templates.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kube-linter

# Go
go install golang.stackrox.io/kube-linter/cmd/kube-linter@latest

# Direct binary (Linux x86_64)
curl -L https://github.com/stackrox/kube-linter/releases/download/v0.8.3/kube-linter-linux.tar.gz \
  | tar -xz && sudo mv kube-linter /usr/local/bin/
```

## Example

```bash
# Lint a directory of manifests with default checks
kube-linter lint ./k8s/

# Lint a rendered Helm chart, fail CI on any finding
helm template my-app ./charts/my-app | kube-linter lint - \
  --format sarif > kube-linter.sarif

# Use a custom config that disables noisy checks and adds org policy
kube-linter lint ./k8s/ --config .kube-linter.yaml
```

## When to use

- You want a *fast, offline, no-cluster-required* policy gate in CI
  for Kubernetes YAML — pre-commit, PR check, or a stage in the
  Helm-chart release pipeline.
- You ship Helm charts and want every release to be linted against
  the rendered output for missing probes, root containers, and
  resource limits.
- You need SARIF output for code-scanning dashboards (GitHub
  code-scanning, GitLab security dashboards) without standing up
  a full admission-controller stack.

## When NOT to use

- You need *runtime* policy enforcement (block a non-compliant pod
  from being created) — use Kyverno, OPA Gatekeeper, or Validating
  Admission Policies; kube-linter is a static checker.
- Your policies are deeply data-driven (cross-object queries, "no
  two Deployments may share a host port across all namespaces") —
  Rego/Kyverno's expression languages are richer than kube-linter's
  YAML predicates.
- You want CIS-Benchmark-style *cluster* posture scanning of a live
  cluster — that's `kube-bench` / `kubescape`, not kube-linter.
