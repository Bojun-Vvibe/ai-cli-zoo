# glasskube

- **Repo:** https://github.com/glasskube/glasskube
- **Version:** v0.26.1
- **License:** [LICENSE](https://github.com/glasskube/glasskube/blob/main/LICENSE) (Apache-2.0)
- **Category:** Kubernetes package manager (Helm alternative)

## What it is

`glasskube` is a CLI + in-cluster operator that installs and updates
Kubernetes packages from a Git-backed central package repository, treating
each package as a versioned `Package` / `ClusterPackage` CRD reconciled by
the operator rather than a client-side `helm template | kubectl apply` blast.
Dependencies are resolved transitively (install `cert-manager`, get the
issuers it needs), updates are surfaced declaratively (`glasskube update`
shows what is out of date and applies the change through the operator), and
a local web UI (`glasskube serve`) gives a browse-and-install view backed by
the same CRDs the CLI drives. Custom and private package repositories are
first-class — point the CLI at any Git repo with the documented manifest
layout and it shows up alongside the public catalog.

## Install

```
brew install glasskube/tap/glasskube                                   # macOS / Linuxbrew
# or grab the binary from https://github.com/glasskube/glasskube/releases
glasskube version
glasskube bootstrap                            # install the in-cluster operator
```

## Basic usage

```
glasskube list                                 # show installed + available packages
glasskube install cert-manager                 # resolve deps, create Package CR
glasskube install ingress-nginx --enable-auto-updates
glasskube update                               # show + apply pending updates
glasskube describe cert-manager                # status, version, dependencies
glasskube uninstall cert-manager               # remove + cascade dependents
glasskube serve                                # local web UI on :8580
glasskube repo add my-team https://github.com/me/glasskube-packages.git
```

## When to use it

- You want **a package manager that behaves like `apt` / `brew` for the
  cluster** — `install <name>`, `update`, `uninstall`, with dependency
  resolution and a single source of truth (the `Package` CR) instead of
  scattered `helm list -A` output across namespaces.
- You want **GitOps-friendly app installs without writing Helm values**
  for every common add-on — the CRD is the desired state, the operator
  reconciles, and a Flux / Argo CD setup can manage `Package` resources
  the same way it manages anything else.
- You want **a browsable local UI for ad-hoc install/upgrade** on a dev
  cluster without standing up Rancher / Headlamp / a full dashboard —
  `glasskube serve` is one command.
- Skip it when **Helm is the lingua franca your team already speaks**
  (charts, values, releases, hooks) and the operator-CRD model is a step
  sideways rather than forward — Helm is not going anywhere, glasskube is
  for teams that find `helm install` ergonomics painful at scale, and skip
  it for **GitOps-only shops** where [`flux`](../flux/) / [`argocd`](../argocd/)
  already own application lifecycle from a Git repo (glasskube can layer in,
  but its UX shines as the primary install path).
