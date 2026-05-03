# vcluster

> **Virtual Kubernetes clusters that run inside a namespace of a
> host cluster** — a single Go binary + lightweight in-cluster
> control plane (k3s / k8s / k0s API server, syncer, scheduler) that
> presents itself to users as a fully-featured Kubernetes cluster
> while reusing the host cluster's nodes for actual pod scheduling.
> Pinned to **v0.34.0** (release published 2026-04-29,
> [LICENSE](https://github.com/loft-sh/vcluster/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/loft-sh/vcluster>.

## TL;DR

`vcluster` is what you reach for when "give every team / PR / tenant
their own Kubernetes cluster" is the right user experience but
spinning up a real EKS / GKE / AKS cluster for each one is
prohibitively slow (10+ minutes), expensive (a control plane bill
per cluster), and operationally heavy (one upgrade per cluster, one
network policy set per cluster). A `vcluster` is a real Kubernetes
API server (k3s by default; optionally k8s, k0s, or eks-distro)
running as a single pod inside one namespace of the host cluster,
with its own etcd (or sqlite, or external db), its own RBAC, its
own CRDs, and its own admission chain — but its workload pods are
synced down into the host namespace and scheduled by the host's
real kubelets. Cluster spin-up is seconds, not minutes; cost is one
pod, not one control plane; isolation is namespace-grade plus an
API surface that pretends to be cluster-grade.

## Install

```bash
# Homebrew
brew install vcluster

# curl install
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/download/v0.34.0/vcluster-darwin-arm64" \
  && sudo install -m 0755 vcluster /usr/local/bin/vcluster

# verify
vcluster --version    # vcluster version 0.34.0
```

The `vcluster` binary is the management CLI; the in-cluster
control plane is installed via Helm chart (`loft-sh/vcluster`) or
the `vcluster create` subcommand which wraps Helm.

## License

Apache-2.0 — see [LICENSE](https://github.com/loft-sh/vcluster/blob/main/LICENSE).
Permissive; the upstream OSS project is fully featured. Loft
(the company) ships a paid platform on top for sleep-mode,
multi-cluster fleet management, and SSO; the OSS `vcluster` binary
itself does not phone home.

## One Concrete Example

```bash
# 1. on a host cluster (kind, EKS, whatever) — create a vcluster
#    inside the namespace `team-a` and immediately switch kube-context
#    into it.
vcluster create alpha --namespace team-a --connect

# 2. you are now talking to the vcluster's API server through a
#    local port-forward; everything you do is scoped to it.
kubectl get nodes
# NAME                STATUS   ROLES    AGE   VERSION
# host-node-1         Ready    <none>   12s   v1.31.2+vcluster
# host-node-2         Ready    <none>   12s   v1.31.2+vcluster

kubectl create namespace billing
kubectl apply -f deployment.yaml

# 3. underneath, the syncer translates the vcluster pods into pods
#    in the host namespace `team-a` with a name prefix:
kubectl --context host-cluster -n team-a get pods
# NAME                                                  READY  STATUS
# my-app-7c9-x42-x-billing-x-alpha                      1/1    Running

# 4. disconnect (your kube-context flips back to the host cluster)
vcluster disconnect

# 5. reconnect later
vcluster connect alpha --namespace team-a

# 6. tear it down — one Helm uninstall, one namespace's worth of
#    state to clean up.
vcluster delete alpha --namespace team-a

# 7. list every vcluster across all namespaces of the host cluster
vcluster list
```

## Niche It Fills

**Per-tenant, per-PR, per-environment Kubernetes clusters at
namespace cost.** A real Kubernetes cluster is a bad unit of
multi-tenancy (CRDs are cluster-scoped, RBAC for namespace
isolation is brittle, cluster-scoped operators conflict), but
spinning up a *real* second cluster per tenant is a bad unit of
cost (control plane fee, separate node pool, separate upgrade
cadence). `vcluster` carves out the sweet spot: each tenant /
PR / environment gets *its own* API server, its own etcd, its own
CRDs, its own admission webhooks, its own audit log — but the
pods land on the host cluster's existing nodes via a syncer that
translates `vcluster-namespace/pod-name` into
`host-namespace/pod-name-x-vcluster-namespace-x-vcluster-name`.
The user experience is "I have a cluster"; the operator experience
is "one Helm chart, one namespace, one bill."

## Why use it

Three concrete things that pay back the operational complexity:

1. **Cluster-scoped resources without cluster-scoped blast
   radius.** A `vcluster` tenant can install their own CRDs (an
   Argo CD CRD set, a Strimzi CRD set, a Crossplane provider) and
   they only exist inside that vcluster. On the host they appear as
   a single ConfigMap of vcluster state. Compare to "give the team
   a namespace" — they cannot install CRDs at all.
2. **Seconds to spin up, no control-plane bill.** `vcluster create`
   completes in 5–15 s on a warm cache; the only resources it
   consumes are one pod (~256 MB RAM with the default k3s
   distro) plus whatever pods the tenant deploys. PR-environment
   pipelines that previously had to share a namespace and rewrite
   every Helm chart with a unique prefix can now do
   `vcluster create pr-${{ github.event.number }}` and treat it
   like a fresh cluster.
3. **Real Kubernetes API, not an emulation.** Because the
   vcluster's control plane is *actually k3s/k8s/k0s* and not a
   shim, every kubectl plugin, every controller, every operator,
   every Helm chart works without modification. Compare to
   "namespace-as-a-tenant" tooling that rewrites resource scopes
   and breaks half the ecosystem.

For an LLM-driven agent that needs to provision a sandbox
Kubernetes environment per task — apply manifests, run kubectl
commands, then throw the whole thing away — `vcluster create
sandbox-${task_id} && vcluster connect ... && ... && vcluster
delete` is a sub-minute lifecycle that costs effectively nothing
on an idle host cluster.

## Vs Already Cataloged

- **Vs [`kind`](../kind/) / [`k3d`](../k3d/):** closest peers in
  the "lightweight Kubernetes for dev/test" niche, but a
  different layer. `kind` and `k3d` run a *whole new cluster
  inside Docker* (each with its own kubelet, network, container
  runtime); `vcluster` runs a *control plane inside an existing
  cluster* and reuses that cluster's nodes for scheduling. Pick
  `kind`/`k3d` for laptop-local dev where you have no host
  cluster; pick `vcluster` for shared CI / staging environments
  where a host cluster already exists and you want cheap tenants
  on top of it.
- **Vs [`talosctl`](../talosctl/) / [`k0sctl`](../k0sctl/):**
  different layer entirely — Talos and k0s are *real cluster*
  installers (provision nodes, install OS-level Kubernetes);
  `vcluster` provisions *virtual* clusters inside one of those
  real clusters. They compose: use `talosctl` to bring up the
  host cluster, then use `vcluster` to carve it into tenants.
- **Vs [`kubectx`](../kubectx/):** orthogonal — `kubectx` switches
  *between* existing kube-contexts; `vcluster` *creates* the
  contexts you switch into. `vcluster connect alpha` writes a
  new entry into `~/.kube/config` that `kubectx alpha` then
  jumps to.
- **Vs [`flux`](../flux/) / [`argocd`](../argocd/):** orthogonal —
  GitOps controllers reconcile manifests *into* a cluster;
  `vcluster` provides the cluster they reconcile into. A common
  pattern is "one Argo CD on the host cluster, one
  `Application` per `vcluster`, each pointing at its own repo
  path."
- **Vs [`mirrord`](../mirrord/):** different problem — `mirrord`
  lets a local process pretend to be inside a remote pod for
  debugging; `vcluster` lets a remote namespace pretend to be a
  whole cluster for tenancy. Both pair well: develop a controller
  with `mirrord` against a `vcluster` so you do not pollute the
  host's API.

## Caveats

- **Pod scheduling still uses the host cluster's nodes, scheduler,
  and resource quota.** A vcluster tenant cannot oversubscribe the
  host; if the host has no capacity, vcluster pods stay Pending.
  Set host-namespace `ResourceQuota` to bound a noisy tenant.
- **Some cluster-scoped resources are *synced*, not isolated.**
  By default PVs, StorageClasses, and IngressClasses are
  passed-through from the host (you can opt into per-vcluster
  storage with the embedded storage backend). Audit which APIs
  your tenants need fully-isolated.
- **Networking depends on the syncer mode.** Pods inside a
  vcluster see vcluster-scoped Services that are translated to
  host Services on egress. Cross-vcluster pod IPs are reachable
  by default (they are all on the host CNI), so isolation is
  API-level, not network-level — pair with NetworkPolicies (or
  Cilium policies) on the host to enforce true east-west
  isolation between tenants.
- **Upgrades happen per-vcluster.** Each vcluster pins its own
  k8s version and its own CRD set; upgrading the host cluster
  does *not* upgrade the vclusters inside it. Treat each
  vcluster as its own upgrade unit.
- **The OSS project is the engine; the SaaS platform is the
  fleet manager.** `vcluster` (OSS) creates and manages
  individual vclusters; `vcluster.pro` / `loft` (paid) layer
  multi-cluster sleep-mode, SSO, cost dashboards, and template
  libraries on top. The OSS binary is fully self-sufficient for
  small fleets; expect to either build your own management UI or
  pay for the platform once you cross ~50 vclusters.
