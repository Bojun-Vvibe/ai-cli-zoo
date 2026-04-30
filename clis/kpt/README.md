# kpt

> **Configuration-as-data toolchain for Kubernetes manifests.**
> Treats YAML packages as the unit of distribution, mutates them
> with composable `function` containers (KRM functions), and
> renders / applies the result with a built-in actuator that
> tracks resource ownership across applies. Pinned to
> **v1.0.0-beta.62**
> ([LICENSE](https://github.com/kptdev/kpt/blob/v1.0.0-beta.62/LICENSE),
> Apache-2.0).

Source: <https://github.com/kptdev/kpt>

## TL;DR

`kpt` reframes Kubernetes config as *data you transform with
typed functions* rather than *templates you string-substitute*.
A *package* is any directory of plain KRM YAML plus a
`Kptfile` declaring upstream source, version, and a `pipeline:`
of mutators / validators. `kpt pkg get <git-url>/path@ref pkg/`
clones an upstream package and records the upstream ref, so
`kpt pkg update pkg/ <new-ref>` performs a real three-way merge
against your local edits — the missing "git pull for manifests"
that Helm and Kustomize do not have. `kpt fn render` walks
`pipeline.mutators` and `pipeline.validators`, executing each
as an OCI container or Starlark / Go-binary function over the
resource list (`set-namespace`, `set-labels`, `apply-replacements`,
`gatekeeper`, `kubeval`, `starlark`, plus your own); functions
are pure (`ResourceList in → ResourceList out`) and composable.
`kpt live apply` is the actuator: it injects an `inventory`
ConfigMap so a follow-up apply can detect and prune resources
the package no longer declares — pruning that `kubectl apply
-f` famously cannot do safely.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kpt

# Direct binary (Linux amd64)
curl -fsSL -o /usr/local/bin/kpt \
  https://github.com/kptdev/kpt/releases/download/v1.0.0-beta.62/kpt_linux_amd64
chmod +x /usr/local/bin/kpt

# Go install
go install -ldflags "-X github.com/kptdev/kpt/run.version=v1.0.0-beta.62" \
  github.com/kptdev/kpt@v1.0.0-beta.62
```

## Example

```bash
# Fetch an upstream package at a pinned ref
kpt pkg get https://github.com/kptdev/kpt.git/package-examples/wordpress@v0.9 wp/

# Author a Kptfile pipeline: namespace + label every resource
cat >> wp/Kptfile <<'YAML'
pipeline:
  mutators:
    - image: gcr.io/kpt-fn/set-namespace:v0.4
      configMap: { namespace: prod }
    - image: gcr.io/kpt-fn/set-labels:v0.2
      configMap: { tier: web, env: prod }
  validators:
    - image: gcr.io/kpt-fn/kubeval:v0.3
YAML

# Render the package (runs the pipeline, writes back in place)
kpt fn render wp/

# Initialise inventory + apply with prune-on-removal semantics
kpt live init wp/
kpt live apply wp/ --reconcile-timeout=5m

# Pull upstream changes with a 3-way merge against local edits
kpt pkg update wp/@v0.10
```

## When to use

- You want **package versioning + 3-way merge updates** for
  Kubernetes manifests — Helm chart upgrades and Kustomize
  overlays do not give you `pkg update` semantics.
- You want declarative, reproducible mutation pipelines where
  each step is a container image with a typed contract — easier
  to audit and unit-test than Go templates or JSONNet.
- You need safe `apply` with **prune-on-delete** without writing
  custom controllers or wrapping `kubectl apply --prune`'s
  label-based footguns.

## When NOT to use

- Your team is fully on Helm and the chart ecosystem
  (`bitnami/*`, `prometheus-community/*`) is your source of
  truth — kpt does not consume Helm charts directly without a
  render-then-import dance.
- You want a controller-based GitOps loop — pair
  [`argocd`](../argocd/) or [`flux`](../flux/) with plain
  Kustomize / Helm; kpt is a *client-side* package + actuator
  toolchain, not a reconciler in the cluster.
- You only have one tiny app — `kustomize build | kubectl apply
  -f -` (or [`kustomize`](../kustomize/) directly) is fewer
  moving parts.
