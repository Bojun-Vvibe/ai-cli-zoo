# clusterctl

> **The Cluster API CLI: declaratively bootstrap, upgrade, and
> tear down Kubernetes clusters using Kubernetes itself as the
> control plane.** Pinned to **v1.13.1**
> ([LICENSE](https://github.com/kubernetes-sigs/cluster-api/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/kubernetes-sigs/cluster-api>

## TL;DR

`clusterctl` is the operator-facing CLI for Cluster API
(CAPI), the Kubernetes-SIG project that turns "spin up a new
production-grade cluster on AWS / Azure / GCP / vSphere /
bare-metal" into a Kubernetes resource (`Cluster`,
`MachineDeployment`, `KubeadmControlPlane`) you `kubectl
apply`. The flow: install `clusterctl init` into a small
"management cluster", then declare workload clusters as YAML
and let CAPI's controllers reconcile them — provisioning VMs,
installing the control plane, joining nodes, rolling upgrades.
`clusterctl` is the bootstrapper, generator, and lifecycle
helper for that loop.

## Install

```bash
# macOS (Homebrew)
brew install clusterctl

# Linux / macOS (binary)
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.13.1/clusterctl-$(uname -s | tr A-Z a-z)-amd64 \
  -o /usr/local/bin/clusterctl && chmod +x /usr/local/bin/clusterctl

# verify
clusterctl version
```

## Examples

```bash
# Turn the current kube-context into a CAPI management cluster
clusterctl init --infrastructure aws

# Generate a workload-cluster manifest (templated by provider)
clusterctl generate cluster prod-east \
  --kubernetes-version v1.31.2 \
  --control-plane-machine-count=3 \
  --worker-machine-count=6 \
  --infrastructure aws > prod-east.yaml
kubectl apply -f prod-east.yaml

# Fetch the workload kubeconfig once provisioning settles
clusterctl get kubeconfig prod-east > prod-east.kubeconfig

# In-place upgrade of CAPI controllers in the management cluster
clusterctl upgrade plan
clusterctl upgrade apply --contract v1beta1

# Move all CAPI objects to a different management cluster (DR drill)
clusterctl move --to-kubeconfig=new-mgmt.kubeconfig
```

## When to choose it

Pick CAPI + `clusterctl` when you operate **many** Kubernetes
clusters across heterogeneous infrastructure and want one
declarative API and one upgrade story across all of them —
typical signals: 5+ clusters, multiple clouds or on-prem +
cloud, a platform team that wants `kubectl apply` to be the
sole interface for provisioning, or a need to script
"reproduce production in a fresh account" as code review on a
YAML diff. Also a good fit when GitOps already owns app
config and you want infrastructure to live in the same repo /
ArgoCD / Flux pipeline.

Skip it for one-off dev clusters — [`kind`](../kind/) or
[`minikube`](../minikube/) are cheaper. Skip it if your
provider has a managed control plane you trust (EKS / GKE /
AKS) *and* you only need to manage that one provider; the
provider's own IaC (CloudFormation, Terraform, Pulumi) is
shorter for the small case. CAPI earns its complexity at
fleet scale.

## Vs adjacent tools

- **Vs Terraform / Pulumi modules for EKS/GKE/AKS:** those
  treat clusters as cloud resources owned by an external
  state backend. CAPI treats clusters as Kubernetes objects
  in a CRD-driven control loop, which composes with GitOps
  and admission policy. Different mental model; CAPI wins
  when "the cluster definition" should live next to "what
  runs on the cluster".
- **Vs [`talosctl`](../talosctl/):** Talos is a specific
  immutable OS + control plane; `talosctl` provisions Talos
  clusters directly. CAPI is provider-pluggable (CAPA / CAPZ
  / CAPG / CAPV / CAPBM / CAPI-Talos…), so you can run Talos
  *under* CAPI for the management surface but mix it with
  non-Talos clusters in the same control plane.
- **Vs [`kops`](https://github.com/kubernetes/kops):** kops
  predates CAPI; AWS-centric, imperative state, mostly in
  maintenance mode. CAPI is the SIG-blessed successor.
- **Vs [`k0sctl`](../k0sctl/) / [`kubespray`](https://github.com/kubernetes-sigs/kubespray):**
  those are Ansible-style "SSH into nodes and configure
  Kubernetes" tools. Great for bare-metal / air-gapped /
  one-cluster setups. CAPI assumes you have an
  infrastructure provider that can mint VMs and a management
  cluster to host the controllers.
- **Vs [`vcluster`](../vcluster/):** orthogonal — `vcluster`
  gives you many *virtual* control planes inside one host
  cluster (cheap multi-tenant dev). CAPI gives you many
  *real* clusters across infrastructure (production fleet).

## Caveats

- **Needs a management cluster.** Bootstrapping is a
  bootstrap problem: you need a small Kubernetes somewhere
  (`kind`, an existing cluster, a managed one) to run
  `clusterctl init` against. Plan for HA on it; if it dies,
  reconciliation stops.
- **Provider quality varies.** CAPA (AWS) and CAPZ (Azure)
  are mature and CNCF-graduated infrastructure; smaller
  providers (some bare-metal, some niche clouds) have
  thinner test coverage and slower release cadence. Check
  the provider's release notes against your CAPI version.
- **`v1beta1` contract is not `v1`.** The CAPI contract is
  still pre-1.0 in places; minor releases occasionally
  require `clusterctl upgrade apply` to re-stamp CRDs. Read
  the release notes before upgrading.
- **`clusterctl move` is for migrations, not backups.** It
  pivots ownership of objects between management clusters in
  a single transaction; it does not give you point-in-time
  recovery. Pair with `velero` or equivalent.
