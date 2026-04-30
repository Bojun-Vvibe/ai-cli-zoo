# flux

> **GitOps continuous-delivery toolkit for Kubernetes.** The CNCF-graduated
> sister-project to Argo CD: a set of Kubernetes controllers (source,
> kustomize, helm, notification, image-automation) plus the `flux` CLI
> that bootstraps the controllers into a cluster and wires
> `GitRepository` / `OCIRepository` / `HelmRepository` / `Bucket`
> sources to `Kustomization` / `HelmRelease` reconcilers. Pinned to
> **v2.8.6**
> ([LICENSE](https://github.com/fluxcd/flux2/blob/v2.8.6/LICENSE),
> Apache-2.0).

Source: <https://github.com/fluxcd/flux2>

## TL;DR

`flux` is the GitOps controller suite where the cluster polls git
(or an OCI registry, or an S3-shaped bucket) and reconciles itself,
with no central CD server: every state mutation is a commit, every
reconciliation is a controller loop, and the CLI mostly bootstraps
or pokes the controllers. `flux bootstrap github --owner=acme
--repository=fleet --path=clusters/prod` writes the controller
manifests to git, applies them to the cluster, and enrolls the
cluster as self-managed (the cluster updates its own controllers on
the next git push). Sources (`GitRepository` etc.) fetch artifacts;
`Kustomization` reconciles a path inside a source against the
cluster, with `dependsOn` chaining (database before app), `prune:
true` deleting removed objects, `healthChecks` blocking on
`Deployment` Ready, and `postBuild.substituteFrom` doing
ConfigMap-driven variable substitution. `HelmRelease` wraps Helm
chart installs with the same source / drift / health semantics. The
image-automation controllers can scan a registry on a tag policy
(`semver: ">=1.0.0"`) and write the new tag back to git, closing the
"new image → updated manifest" loop without a human PR.

## Install

```bash
# Homebrew (macOS / Linux)
brew install fluxcd/tap/flux

# Direct binary install
curl -s https://fluxcd.io/install.sh | sudo FLUX_VERSION=2.8.6 bash

# Bootstrap into a cluster against a GitHub repo
export GITHUB_TOKEN=ghp_...
flux bootstrap github \
  --owner=acme --repository=fleet \
  --branch=main --path=clusters/prod \
  --personal
```

## Example

```bash
# Verify cluster prerequisites and existing install
flux check --pre
flux check

# Register a git source and reconcile a path against the cluster
flux create source git acme-apps \
  --url=https://github.com/acme/apps.git \
  --branch=main --interval=1m
flux create kustomization web \
  --source=GitRepository/acme-apps \
  --path=./web/overlays/prod \
  --prune=true --interval=10m \
  --health-check-timeout=2m

# Force an immediate sync, watch reconciliation
flux reconcile kustomization web --with-source
flux get kustomizations --watch

# Suspend / resume during an incident
flux suspend kustomization web
flux resume kustomization web
```

## When to use

- You want pull-based GitOps with no central CD server: every
  cluster reconciles itself, multi-cluster fleets scale by adding
  clusters not by scaling a control plane.
- You need first-class OCI-as-source (Helm OCI, OCI artifacts) so
  release artifacts live in your registry, not git LFS.
- You want image-automation closing the loop from registry tag
  policy → manifest commit → cluster sync without a human PR.

## When NOT to use

- You want a UI-driven operator workflow with multi-cluster app
  views and SSO out of the box — [`argocd`](../argocd/) ships the
  better web UI; flux is intentionally controller-shaped + CLI-only.
- You have not standardised on Kustomize / Helm — the controllers
  reconcile those two source types only (no Jsonnet / CDK8s
  rendering inside the cluster; render in CI then commit).
- You only run one cluster and one app — a plain `kubectl apply -k`
  in CI is simpler than running five controllers to reconcile it.
