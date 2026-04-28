# kustomize

- **Repo:** https://github.com/kubernetes-sigs/kustomize
- **Version:** kustomize/v5.8.1 (released 2026-02-09)
- **License:** Apache-2.0 ([LICENSE](https://github.com/kubernetes-sigs/kustomize/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install kustomize` · `go install sigs.k8s.io/kustomize/kustomize/v5@latest` · prebuilt binaries on the GitHub release page · also bundled in `kubectl` as `kubectl kustomize` (the standalone CLI tracks ahead of the bundled version) · binary name is `kustomize`

## Overview

`kustomize` is a **template-free** Kubernetes manifest
customization tool. Where Helm asks you to wrap YAML in a
Go-template DSL (curly braces, `if`/`range`, escaping rules,
chart values), `kustomize` keeps the YAML as YAML and layers
**overlays** on top: a `base/` directory containing the
canonical manifests, plus per-environment overlays
(`overlays/staging/`, `overlays/prod/`) declared as
`kustomization.yaml` files that point at the base and apply
strategic-merge patches, JSON-6902 patches, name prefixes,
namespace overrides, label/annotation injection, image-tag
swaps, and ConfigMap/Secret generators.

The killer property is that every manifest in the source
tree is a **valid Kubernetes object on its own** — a YAML
schema validator works, an editor's YAML LSP works, `kubectl
diff -k overlays/staging` shows real diffs against the
cluster. There is no "render the chart first to see what
actually applies" step; `kustomize build overlays/prod`
produces the final concatenated YAML deterministically from
the inputs, and the same logic is what `kubectl apply -k`
runs in-cluster.

Generators (`configMapGenerator`, `secretGenerator`) hash the
content into the resource name (`my-config-abc123`), so a
ConfigMap value change automatically rolls the consuming
Deployment via the changed name reference — a built-in
solution to the "ConfigMap update doesn't restart Pods"
problem that Helm needs annotations + tooling to solve.

## Use cases

- **Multi-environment manifest management** without learning
  Helm's templating DSL — base + overlays per env covers 80%
  of the use cases Helm is reached for.
- **GitOps with Argo CD or Flux.** Both speak `kustomize`
  natively; an `Application` / `Kustomization` resource
  pointing at an overlay path renders and applies on every
  sync.
- **Patching third-party YAML you don't control.** Pull an
  upstream `nginx-ingress` / `cert-manager` / `metrics-
  server` install YAML as a `resources:` entry, then patch
  what you need (replicas, resource limits, node selectors,
  image registry mirror) without forking the upstream chart.
- **In-cluster rendering via `kubectl apply -k`** — no extra
  binary on the deploy host; any `kubectl` 1.14+ already
  ships a (sometimes-older) `kustomize` build path.

## Why pick this

Pick `kustomize` when the manifests are simple enough that
templating is overkill but you still need per-environment
variation — i.e. most internal services. The
"YAML stays YAML, overlays patch it" model is dramatically
easier to onboard a new engineer onto than Helm's chart
authoring story, and the lack of a values.yaml indirection
layer makes "what actually gets applied" a one-command
question (`kustomize build`). Pick Helm instead when you are
*publishing* a reusable component for others to install
(Helm's values + dependencies model is the right shape for
distributable charts). The two also compose: `helm template
| kustomize` is a common pattern for patching upstream
charts.

## Comparable alternatives

- `helm` — chart-as-package; pick for distributed reusable
  components, complex parameterization, or release lifecycle
  management.
- `jsonnet` / `tanka` — programmatic manifest generation;
  pick when you need real conditionals, loops, and shared
  libraries across hundreds of services.
- `cdk8s` — TypeScript / Python / Go / Java manifests as
  code; pick when your team is already fluent in one of
  those and YAML editing is the friction point.
- `ytt` (Carvel) — alternative templating with stronger
  schema validation and overlays; smaller community than
  Helm/kustomize.
- `kpt` — sigs.k8s.io's package-and-pipeline tool; uses
  kustomize-style functions for transformation, broader
  scope (lifecycle, hydration, validation pipelines).
- Plain `envsubst` + `kubectl apply` — minimal; works for
  one-variable substitutions, breaks down past that.
