# nomad

- **Repo:** https://github.com/hashicorp/nomad
- **Version:** 2.0.0 (tagged 2026-04-21; first 2.x GA; 1.x line continues for LTS users on 1.10.x)
- **License:** BUSL-1.1 (Business Source License 1.1; converts to MPL-2.0 four years after each release; the SDK / API client packages remain MPL-2.0) — see [`LICENSE`](https://github.com/hashicorp/nomad/blob/main/LICENSE)
- **Language:** Go (single static binary; the same `nomad` binary runs as server, client, or CLI by argument)
- **Install:** `brew install hashicorp/tap/nomad` · `apt install nomad` (HashiCorp APT repo) · `dnf install nomad` (HashiCorp YUM repo) · `pacman -S nomad` · or download `nomad_2.0.0_<os>_<arch>.zip` from https://releases.hashicorp.com/nomad/2.0.0/ (signed `_SHA256SUMS.sig` published alongside)

## Overview

`nomad` is a single-binary cluster scheduler that runs
**any workload type** — Linux / Windows containers
(Docker, podman, containerd, exec), raw binaries, JVM
jars, QEMU VMs, WASM modules, and arbitrary plugin task
drivers — across a fleet of servers (Raft-replicated
control plane) and clients (worker nodes that report
fingerprinted resources: CPU model, GPU PCI IDs,
attached EBS volumes, kernel version). Job specs are
HCL2 documents declaring `group { task { driver =
"docker" config { image = "..." } resources { cpu = ...
memory = ... } } count = N }`; the bin-packing scheduler
places allocations across clients, the orchestrator
keeps `count` running, and `nomad job run` /
`nomad deployment status` give canary / blue-green /
rolling rollouts with automatic revert on health-check
failure. Where Kubernetes is "container scheduler with
a 200-CRD ecosystem", Nomad is "scheduler core, every
adjacent concern is a separate HashiCorp tool you opt
into" — `consul` for service discovery + service mesh,
`vault` for secrets, `terraform` for cluster
provisioning. The result: a 100-node cluster runs on
three 2 GB Raft servers and N clients with
single-digit-MB RAM overhead per node, and the entire
control plane is one binary you can `scp` and run.

## Niche

**Single-binary multi-driver workload scheduler that
runs containers, raw binaries, VMs, and WASM with one
HCL job spec, with a Raft control plane small enough
for a homelab and large enough for ~10k-node
production fleets.** The role is "the scheduler you
reach for when Kubernetes' surface area is wrong for
the problem — too small (one team, mixed workload
types) or too operationally heavy (no platform team to
own kube). The competing universe is `kubernetes` (via
`kubectl` / `k3s` / `k3d` / `kind`) plus, at the
homelab edge, `docker swarm` and bespoke
systemd-on-N-hosts setups — see comparisons below.

## When to use

- You have **mixed workload types**: containers + a
  legacy JVM jar + a Windows binary + a one-off QEMU VM
  for a vendor appliance, and you want one scheduler
  for all of it instead of "Kubernetes for the new
  stuff and ad-hoc systemd for the legacy stuff".
- You want a control plane that **fits on three 2 GB
  servers** and a client agent under 100 MB RSS — Nomad
  servers run Raft with a ~10 MB working set per
  thousand allocations; the client agent is a thin
  fingerprint-and-execute loop.
- You are on Windows / FreeBSD / illumos and Kubernetes
  support is second-class — Nomad's client agent runs
  natively on all three with full feature parity for
  the `exec` / `raw_exec` / `java` / `docker` (where
  available) drivers.
- You want **batch / system / service / parameterised /
  periodic** as job-type primitives without installing
  CronJob CRDs and a job-controller — `type =
  "batch"` and `periodic { cron = "0 3 * * *" }` are
  built-in.
- You want **service mesh + intentions + L7 routing**
  via `consul-connect` sidecars driven by a one-line
  `connect { sidecar_service {} }` block in the job
  spec, with mTLS issued from the Consul / Vault PKI;
  no Istio control plane to operate.
- You want **deployments with health-checked canaries +
  automatic revert** as a first-class job-spec field
  (`update { canary = 1, auto_revert = true,
  healthy_deadline = "2m" }`) instead of a separate
  Argo / Flagger install.

## When NOT to use

- Your team / platform is already invested in the
  Kubernetes ecosystem (Helm, Argo, Flux, Istio,
  Prometheus Operator, KEDA, kubeflow, the entire CRD
  galaxy) — Nomad does not interop with kube CRDs;
  switching costs are real. Pick `kubernetes` /
  `k3s` / `kind` if the team already speaks kube.
- You need **the largest ecosystem of off-the-shelf
  Operators** (Postgres, Kafka, Redis, Cert-Manager,
  External-DNS) wired into one control plane — that
  ecosystem only exists for Kubernetes. Nomad job
  specs for those workloads exist but are
  community-maintained and shallower.
- The BUSL-1.1 licence terms are a hard blocker for
  your distribution model — you cannot offer a
  commercial Nomad-as-a-service competing with
  HashiCorp Cloud Platform without a separate licence.
  The OSS use case (run it yourself) is fine.
- You need **fully-managed scheduler-as-a-service** on
  the same SLO as EKS / GKE / AKS — HCP Nomad exists
  but the managed-Kubernetes ecosystem is broader.

## Comparison vs alternatives in zoo

- `kubernetes` (via [`k3d`](../k3d/) / [`k9s`](../k9s/)
  / [`kustomize`](../kustomize/) / [`kpt`](../kpt/) /
  [`helm`](../helm/) / [`helmfile`](../helmfile/) /
  [`argocd`](../argocd/) / [`flux`](../flux/) /
  [`argo-rollouts`](../argo-rollouts/)) — the
  industry-default scheduler with the largest
  ecosystem. Pick kube when the team already speaks it
  or you need the CRD galaxy; pick Nomad when the
  control-plane footprint, mixed-driver story, or
  HCL-vs-YAML ergonomics matter more than ecosystem
  depth.
- [`consul`](../consul/) — service discovery, KV, and
  service-mesh control plane. Complementary — Nomad
  registers services into Consul automatically when
  `service { provider = "consul" }` is set; Consul
  Connect sidecars are the recommended mesh.
- [`tofu`](../tofu/) (OpenTofu fork of Terraform) /
  [`pulumi`](../pulumi/) — IaC for provisioning the
  hosts Nomad clients run on. Complementary — Nomad
  schedules workloads onto already-provisioned hosts;
  it does not manage the underlying hosts.
- [`skaffold`](../skaffold/) / [`tilt`](../tilt/) —
  local kube-dev loops. Pure kube-side; Nomad's
  equivalent is `nomad job run -dev` against a single
  `nomad agent -dev` process.
- [`kubescape`](../kubescape/) /
  [`kube-score`](../kube-score/) /
  [`kubeconform`](../kubeconform/) — Kubernetes
  policy / posture / schema linters. No Nomad
  equivalent in the zoo at this snapshot; Nomad's
  policy story is Sentinel (HashiCorp Enterprise) or
  community OPA gatekeeping.
- [`vault`](https://github.com/hashicorp/vault) — not
  in the zoo at this snapshot; Nomad's
  `vault { policies = [...] }` block requests
  short-lived dynamic secrets that Vault rotates per
  allocation.

## Why it earns a slot in an AI-native workflow

LLM workloads are heterogeneous in ways Kubernetes
handles awkwardly: an inference fleet wants
`vllm`-in-a-container with GPU device plugins, a
fine-tune wants a raw torchrun process with
host-namespace networking and an NVIDIA driver pinned
to a specific kernel module, a vector-store backfill
wants a one-off batch job with a 12-hour deadline,
and a model-eval harness wants periodic execution
against a moving model registry. Nomad's
multi-driver, multi-job-type model handles all four
in one control plane: `driver = "docker"` for vLLM
with `device "nvidia/gpu" { count = 2 }`, `driver =
"exec"` for the raw torchrun, `type = "batch"` with
`reschedule { unlimited = false }` for the backfill,
`type = "periodic"` for the eval. The HCL
`device "nvidia/gpu"` block plus the
`nvidia-container-runtime` task driver gives
GPU-fingerprinted scheduling without installing the
NVIDIA device plugin, the GPU operator, MIG manager,
and friends. For a small AI team operating its own
fleet, "three 2 GB Nomad servers + N GPU client
nodes" is a tractable platform footprint that does
not require a dedicated kube-ops engineer.

## Example invocations

```bash
# Single-binary dev cluster — server + client in one process
nomad agent -dev

# Production-shape: three Raft servers + N clients
nomad agent -config=/etc/nomad.d/server.hcl     # on each of 3 servers
nomad agent -config=/etc/nomad.d/client.hcl     # on each worker

# Submit, monitor, and roll a job
nomad job init -short                            # writes example.nomad.hcl
nomad job validate example.nomad.hcl
nomad job plan example.nomad.hcl                 # dry-run with diff
nomad job run example.nomad.hcl
nomad status                                     # cluster overview
nomad job status example                         # per-job allocation table
nomad alloc logs -f <alloc-id>                   # tail allocation logs

# Canary deploy then promote
nomad job run -check-index 7 example.nomad.hcl
nomad deployment status <deploy-id>
nomad deployment promote <deploy-id>             # promote canaries to stable

# Run a one-off batch job with a deadline
nomad job dispatch -meta input=s3://bucket/job-42 backfill

# Periodic (cron) job — daily 03:00 UTC eval sweep
cat > eval.nomad.hcl <<'EOF'
job "nightly-eval" {
  type = "batch"
  periodic {
    cron             = "0 3 * * *"
    prohibit_overlap = true
  }
  group "g" { task "t" {
    driver = "docker"
    config { image = "ghcr.io/me/eval:latest" }
    resources { cpu = 4000, memory = 16384 }
  }}
}
EOF
nomad job run eval.nomad.hcl

# Inspect a node's fingerprint (CPU, GPU, memory, kernel)
nomad node status -verbose <node-id>

# ACL bootstrap + token issue
nomad acl bootstrap                              # one-time, captures root token
nomad acl token create -name=ci -policy=deploy

# Drain a node before maintenance, then re-enable
nomad node drain -enable -deadline 1h <node-id>
nomad node drain -disable <node-id>
```
