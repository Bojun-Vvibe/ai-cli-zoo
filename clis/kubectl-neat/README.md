# kubectl-neat

> **Strip the noise from `kubectl get -o yaml`** — a `kubectl`
> plugin that removes server-generated, default, and runtime fields
> from Kubernetes resource dumps so you can read, diff, or re-apply
> them like the manifests you originally wrote. Pinned to **v2.0.4**
> ([LICENSE](https://github.com/itaysk/kubectl-neat/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/itaysk/kubectl-neat>

## TL;DR

`kubectl get pod foo -o yaml` returns ~200 lines for a pod whose
original manifest was 20 — the API server folds in
`status:`, `metadata.managedFields`, `metadata.creationTimestamp`,
`metadata.uid`, `metadata.resourceVersion`, the
`kubectl.kubernetes.io/last-applied-configuration` annotation,
default values for every unset field, and the runtime fields
injected by admission controllers. `kubectl-neat` post-processes
that YAML/JSON and removes everything that wasn't part of your
original intent, leaving a manifest you can paste into a PR, diff
against your gitops repo, or `kubectl apply -f` on another cluster.
It's a single Go binary, installed via `krew` or `go install`, and
chained as `kubectl get … -o yaml | kubectl neat`.

## Install

```bash
# krew (recommended — auto-updates with kubectl plugins)
kubectl krew install neat

# Homebrew
brew install kubectl-neat

# Go
go install github.com/itaysk/kubectl-neat@v2.0.4

# from a release binary
curl -L -o kubectl-neat.tar.gz \
  https://github.com/itaysk/kubectl-neat/releases/download/v2.0.4/kubectl-neat_darwin_arm64.tar.gz
tar xf kubectl-neat.tar.gz
sudo install kubectl-neat /usr/local/bin/

# verify
kubectl neat version    # 2.0.4
```

## License

Apache-2.0 — see
[LICENSE](https://github.com/itaysk/kubectl-neat/blob/master/LICENSE).
Permissive, patent grant included.

## One Concrete Example

```bash
# 1. clean up a single live resource
kubectl get pod my-app-7c8d -o yaml | kubectl neat
# → ~25 lines instead of ~220, no status/managedFields/uid/etc.

# 2. neat-in-place (kubectl-neat understands the full -o pipeline)
kubectl neat get pod my-app-7c8d -o yaml
# same result, one command instead of a pipe

# 3. recover a usable manifest from a cluster you don't own
kubectl get deploy/frontend -o yaml | kubectl neat > frontend.yaml
# diff that against your gitops repo to find drift
diff frontend.yaml gitops/frontend.yaml

# 4. neat a whole namespace into a directory of clean manifests
for kind in deploy svc cm secret ingress; do
  kubectl get $kind -n prod -o yaml | kubectl neat \
    > "exported/${kind}.yaml"
done

# 5. JSON pipeline (for jq / yq / scripts)
kubectl get pod my-app -o json | kubectl neat -o json | \
  jq '.spec.containers[].image'

# 6. neat an arbitrary file (not just live resources)
cat dump.yaml | kubectl neat -f -
```

## Niche It Fills

**The "round-trip" gap in kubectl.** `kubectl get -o yaml` was
designed to be lossless for the API server, not readable for the
operator: it includes every default, every status field, every
admission-controller mutation, and the entire `managedFields`
graph used by server-side apply. `kubectl-neat` exists because
the *human* use case — "show me what was actually declared" — is
distinct from the *server* use case, and there is no flag on
`kubectl` to express it. The result is the difference between a
manifest you can read in 10 seconds and one you have to scroll
through for a minute looking for the three lines that matter.

## Why use it

1. **Drift detection without parsing managedFields.** Diffing
   a live resource against your gitops source-of-truth is the
   single most common "is prod what I think it is" question;
   `kubectl-neat` makes the diff signal-only instead of buried
   in 180 lines of server bookkeeping.
2. **Cluster-to-manifest export when you didn't write the
   originals.** Inheriting a cluster (acquisition, on-call,
   archeology) — `kubectl-neat` reconstructs the manifests you
   would have written, ready to commit to a fresh repo.
3. **Pure stdin/stdout filter.** Composes with `jq`, `yq`,
   `kubectl diff`, `kustomize`, `helm template`, `argocd app
   diff`, etc. without any plugin contract — it's just YAML in,
   YAML out.

For an LLM-CLI workflow that asks the model to reason about a
Kubernetes resource, piping through `kubectl-neat` first cuts
the prompt token cost ~5–10× and removes fields the model would
otherwise hallucinate as significant.

## Vs Already Cataloged

- **Vs [`kustomize`](../kustomize/):** orthogonal — `kustomize`
  is a *forward* tool (overlay manifests → final YAML);
  `kubectl-neat` is a *reverse* tool (cluster YAML → readable
  manifest). Often paired: neat a live resource, then turn the
  diff into a kustomize overlay.
- **Vs [`kubectl-tree`](../kubectl-tree/):** different verbs.
  `kubectl-tree` shows the *ownership graph* of resources
  (Deployment → ReplicaSet → Pods); `kubectl-neat` cleans the
  *content* of a single resource. Use both when reverse-
  engineering an unknown cluster.
- **Vs [`kube-score`](../kube-score/) / [`kubeconform`](../kubeconform/):**
  those *evaluate* manifests against best-practice rules or the
  Kubernetes schema. `kubectl-neat` doesn't judge — it just
  removes noise. Run neat first, then feed the clean manifest
  to a linter.
- **Vs `kubectl diff` (built-in):** `kubectl diff` shows what
  *would* change if you applied a file; it doesn't help when
  you want to *export* the live state. `kubectl-neat` is the
  missing other half.

## Caveats

- **Heuristic, not authoritative.** `kubectl-neat` removes fields
  it *believes* are server-generated based on a curated list and
  the OpenAPI schema. It can occasionally strip a field you set
  to its default value on purpose (e.g. `imagePullPolicy:
  IfNotPresent`); review the diff before treating the output as
  canonical source.
- **Doesn't preserve `last-applied-configuration` round-trips.**
  If you neat a resource and re-apply with `kubectl apply`,
  server-side apply will re-establish ownership but client-side
  apply may see the resource as "new". Use server-side apply
  (`kubectl apply --server-side`) when round-tripping neat'd
  manifests.
- **Status block is *always* removed.** That's the point, but if
  you wanted to capture the status (e.g. for a snapshot of a
  failure), pipe through `yq '.status'` *before* neat instead.
- **CRDs without OpenAPI schemas get a coarser pass.** For
  custom resources whose CRD doesn't declare defaults, neat
  falls back to the generic field list — it strips the obvious
  Kubernetes-wide noise (`uid`, `resourceVersion`, timestamps)
  but can't know which CRD-specific fields are defaults.
- **Maintenance cadence is occasional.** Last release v2.0.4
  shipped mid-2024. The tool is stable for the field set it
  knows about; newer Kubernetes API additions (e.g. fields
  added in 1.30+) may not yet be in the strip list. PRs are
  accepted but landings are slow.
