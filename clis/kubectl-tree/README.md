# kubectl-tree

- **Repo:** https://github.com/ahmetb/kubectl-tree
- **Version:** v0.6.0 (latest stable, released 2026-03-24)
- **License:** Apache-2.0 ([LICENSE](https://github.com/ahmetb/kubectl-tree/blob/master/LICENSE))
- **Language:** Go
- **Install:** `kubectl krew install tree` · `go install github.com/ahmetb/kubectl-tree@latest` · prebuilt binaries on the GitHub release page · invoked as `kubectl tree`

## What it does

`kubectl tree` is a `kubectl` plugin that renders the **owner-reference
graph** of a Kubernetes object as an ASCII tree. Pick any object —
a `Deployment`, a `StatefulSet`, a `CronJob`, a Helm release marker
ConfigMap, an Argo CD `Application`, a Crossplane `Composite`, or a
custom CRD — and it walks downward through `metadata.ownerReferences`
to find every resource the controller chain spawned: ReplicaSets,
Pods, Jobs, PVCs, Services, Endpoints, generated ConfigMaps /
Secrets, and the same recursion across CRDs you do not have to
teach it about (it discovers them via the API server's discovery
endpoint).

The output is a single screen showing the whole controller-spawned
subtree with each row tagged by `KIND/NAME` and the namespace
inherited from the root object. `--show-group` adds the API group
column for disambiguation when two CRDs use the same `Kind`,
`-A` / `--all-namespaces` widens the search when the controller
lives in one namespace and writes children into another, and
`--show-images` decorates Pod rows with the container images they
are running so a "what's actually deployed under this Application"
question is one command, not five.

## When to pick it / when not to

Reach for `kubectl tree` whenever a controller is involved and the
question is "what did *this* root object actually create." That
covers Helm releases (point at the release Secret / ConfigMap and
see every spawned object), Argo CD `Application`s (the Application
owns the Kustomization output), Crossplane Compositions (the XR
owns its composed resources), Operators (the CR owns the
StatefulSets / Services / PVCs the operator generated), and even
plain `Deployment` → `ReplicaSet` → `Pod` triage where you want
the inactive ReplicaSets visible too. It is also the fastest way
to find orphaned children whose owner has been deleted but whose
finalizers are still holding them in `Terminating`.

Skip it when the relationship you care about is **not** owner-ref
based: NetworkPolicy → Pod by selector, Service → Pod by selector,
ConfigMap mounted by Pod by name. None of those go through
`ownerReferences`, so `kubectl tree` will not see the link. For
selector-driven graphs use `kubectl get pods -l <selector>` or a
visualizer like `k8sviz`. Skip it on very large clusters where
an unscoped `kubectl tree -A <Kind> <name>` would touch every
namespace's discovery endpoints — pin the namespace.

## Why it matters in an AI-native workflow

Coding agents that operate on Kubernetes tend to apply a high-level
object (a Helm chart, a Custom Resource, an Argo Application) and
then need a deterministic way to read what the cluster *actually
materialized* before deciding the next step. `kubectl tree
<Kind> <name> -o wide` returns a bounded, hierarchically grouped
view of exactly the children that root produced — small enough to
fit a context window, structured enough that the agent can reason
about which child failed without re-asking "and what owns that
Pod?" three times. It is also a good post-apply diff signal:
capture the tree before and after, and the set difference is the
controller's actual side-effect.

## Example invocations

```bash
# Walk a Deployment downwards
kubectl tree deploy api

# Across all namespaces (controller in ns A, child in ns B)
kubectl tree -A application argocd-myapp

# Show API group + container images per Pod
kubectl tree deploy api --show-group --show-images

# Trace a Helm release's owned objects
kubectl tree secret sh.helm.release.v1.my-release.v3 -n my-release

# Trace a Crossplane composite resource and its composed children
kubectl tree xpostgresqlinstance my-db --show-group
```

## Alternatives in this catalog

- [`k9s`](../k9s/) — interactive TUI; good for live triage but the
  owner-graph view is a separate plugin and not the default.
- [`kubectx`](../kubectx/) — context / namespace switcher; pairs
  with `kubectl tree` for "switch to this cluster, then walk this
  object's children."
- [`stern`](../stern/) — once `kubectl tree` tells you which Pods
  a controller spawned, `stern <name-prefix>` follows their logs.
- [`kubectl-ai`](../kubectl-ai/) — natural-language → `kubectl`
  translator; can call `kubectl tree` as one of its tool actions.
