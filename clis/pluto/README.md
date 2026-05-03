# pluto

> **Scans Kubernetes manifests, Helm charts, live cluster state, and
> Helm release history for use of *deprecated* and *removed* API
> versions (`extensions/v1beta1 Ingress`, `policy/v1beta1
> PodSecurityPolicy`, `autoscaling/v2beta1`, …) and tells you exactly
> which target Kubernetes version will break them.** Pinned to
> **v5.24.0**, Apache-2.0
> ([LICENSE](https://github.com/FairwindsOps/pluto/blob/master/LICENSE)).

- **Repo:** https://github.com/FairwindsOps/pluto
- **Latest version:** v5.24.0
- **License:** Apache-2.0 (`LICENSE` at repo root)
- **Category:** `kubernetes` / `upgrade-safety` / `api-deprecation` / `helm-lint`
- **Language:** Go

## What it does

Every Kubernetes minor release graduates some APIs to GA, deprecates
others, and *removes* the ones that have been deprecated long enough.
Removal is not a warning — it is a hard 404 at the apiserver. A
`kind: Ingress` manifest pinned to `extensions/v1beta1` was fine on
1.18, deprecated on 1.19, and *gone* on 1.22; a chart pinned to
`policy/v1beta1 PodSecurityPolicy` works on 1.24 and explodes on
1.25. `pluto` solves the "what will break when I upgrade" question
mechanically. It carries an embedded, versioned dataset of every
known deprecation/removal across upstream Kubernetes plus several
add-ons (`cert-manager`, `istio`), and four scanners that all share
the same dataset: (1) `pluto detect-files <dir>` walks a directory of
raw manifests; (2) `pluto detect-helm` scans installed Helm 3
releases by reading their stored manifests from cluster secrets;
(3) `pluto detect-api-resources` queries a live cluster's etcd-backed
state via the apiserver (catches resources created without a manifest
in git); (4) `helm template ... | pluto detect -` is the unix-pipe
form for a CI gate against unreleased charts. Output is human, JSON,
YAML, Markdown, or CSV; `--target-versions k8s=v1.31.0` filters the
report to "what will break on *this* upgrade."

## Install

```bash
# macOS / Linux via Homebrew
brew install fairwindsops/tap/pluto

# All platforms — release binary
curl -L https://github.com/FairwindsOps/pluto/releases/download/v5.24.0/pluto_5.24.0_linux_amd64.tar.gz \
  | tar -xz -C /usr/local/bin pluto

# Helm plugin form (so it lives next to your other helm tooling)
helm plugin install https://github.com/FairwindsOps/pluto
```

## Examples

```bash
# Scan a directory of manifests (CI gate before kubectl apply)
pluto detect-files -d ./manifests --target-versions k8s=v1.31.0

# Scan everything currently installed in the cluster via Helm 3
pluto detect-helm -A --output wide

# Scan live API resources (catches drift not represented in git)
pluto detect-api-resources -A

# Pipe in `helm template` output for an unreleased chart
helm template ./mychart | pluto detect -

# CI failure mode: exit non-zero on any *removed* API
pluto detect-files -d ./manifests --ignore-deprecations \
  --target-versions k8s=v1.31.0 --output json
```

## Why it matters in an AI-native workflow

LLM agents asked to write Kubernetes manifests reliably reach for
whatever API version dominated their training-data cutoff, which is
often the *deprecated* one. `apiVersion: extensions/v1beta1` for an
Ingress, `apiVersion: policy/v1beta1` for a PodSecurityPolicy, and
`apiVersion: autoscaling/v2beta1` for an HPA all show up regularly
in agent-generated YAML and all are removed in current upstream
Kubernetes. Static schema validation ([`kubeconform`](../kubeconform/),
[`kube-linter`](../kube-linter/)) catches *malformed* manifests but
will happily approve a syntactically correct manifest pinned to a
removed API. `pluto` is the missing layer: a deterministic gate
that says "this manifest will be rejected by a 1.31 apiserver,
here is the replacement apiVersion." Pairs with
[`kubeconform`](../kubeconform/) (schema validity, orthogonal),
[`kube-linter`](../kube-linter/) (security/best-practice rules),
[`kube-score`](../kube-score/) (production-readiness scoring), and
[`helm`](../helm/) (the templating layer pluto consumes). The
typical CI pipeline runs all four against the same `helm template`
output: kubeconform for "is it valid YAML against the OpenAPI
schema", pluto for "will the target cluster version still accept
it", kube-linter for "is it safe", kube-score for "is it
production-grade." Orthogonal to runtime tools like
[`popeye`](../popeye/) which scan a *live* cluster for misconfig
rather than upgrade risk.
