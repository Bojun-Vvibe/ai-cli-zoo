# minikube

> **Single-binary local Kubernetes cluster for development and CI.**
> A Go binary (`minikube`) that boots a real, conformant Kubernetes
> cluster on a developer laptop using one of several drivers (Docker,
> Podman, HyperKit, KVM2, VirtualBox, QEMU, VMware, Parallels) and
> exposes the standard `kubectl` API surface. Pinned to **v1.38.1**
> (released 2026-02-19, SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/kubernetes/minikube/blob/master/LICENSE)).

Source: <https://github.com/kubernetes/minikube>

## Repo

- URL: <https://github.com/kubernetes/minikube>
- Owner/org: kubernetes (Kubernetes SIGs, CNCF graduated project umbrella)
- License file: [LICENSE](https://github.com/kubernetes/minikube/blob/master/LICENSE)

## Version

`v1.38.1` — released 2026-02-19. Verify with `minikube version`.
Bundles `kicbase` v0.0.50 (the base node image used by the default
docker / podman drivers) and tracks upstream Kubernetes releases
within ~1–2 weeks of GA — `minikube start --kubernetes-version=v1.34.x`
selects the exact control-plane version, so a single binary can boot
any of the last ~10 minor releases for compatibility testing.

## License

**Apache-2.0** — OSI-approved, permissive. Safe to embed in CI
runner images, ship as part of an internal developer-platform
bootstrap, or vendor into a corporate dev-environment installer.
The kicbase node image and bundled addons are independently
licensed (mostly Apache-2.0 / MIT / BSD); `minikube addons list`
shows the catalog.

## What it does

`minikube start` provisions a node (a container with the docker
driver, a VM with hypervisor drivers), bootstraps kubeadm-installed
control-plane components inside it, writes a kubeconfig context
into `~/.kube/config`, and exits. From that point `kubectl get
nodes` shows a real cluster with a real API server, scheduler,
controller-manager, kubelet, and CNI — the same code paths that
run in production, not a mock. `minikube addons enable ingress |
metrics-server | registry | gvisor | volumesnapshots | …` flips
the optional components on. `minikube tunnel` exposes
`LoadBalancer` Services on the host. `minikube image load` pushes
a locally-built container image into the cluster's runtime
without a registry round-trip — the workflow that makes
inner-loop development tolerable.

Multi-node (`--nodes=3`), multi-cluster (named profiles), HA
control-plane (`--ha`), and multiple container runtimes (containerd,
cri-o, docker) are supported, so the local cluster topology can
match whatever the production cluster looks like.

## When to use

- **Inner-loop Kubernetes development.** A single laptop needs a
  real cluster to iterate on Helm charts, operator code,
  admission webhooks, CRDs, or anything that touches
  cluster-scoped resources. minikube boots in 30–90 s and
  resets in seconds.
- **Per-PR ephemeral CI clusters.** A GitHub Actions / GitLab CI
  job runs `minikube start --driver=docker --nodes=2`, applies
  the chart, runs end-to-end tests against the real API server,
  and tears down. Faster and more representative than mocking
  the API.
- **Compatibility matrix testing.** `--kubernetes-version=…`
  lets one repo's CI verify that an operator works on
  Kubernetes 1.30 / 1.31 / 1.32 / 1.33 / 1.34 in parallel jobs.
- **Trying upstream features pre-GA.** alpha / beta feature
  gates flip via `--feature-gates=...` without waiting for a
  managed-Kubernetes provider to expose them.
- **Workshops and training.** A repeatable `minikube start`
  recipe gets a room of attendees onto identical clusters in
  minutes, without cloud accounts or shared infrastructure.

## When NOT to use

- **You only need to apply a manifest to a real cluster you
  already have.** Skip the local cluster, point `kubectl` at it.
- **You want a long-running shared dev cluster.** Use
  [`vcluster`](../vcluster/) on top of a real cluster, or
  [`kind`](../kind/) on a beefy CI host with multiple profiles.
- **You want to test cloud-provider integrations** (AWS LB
  controller against real ELBs, GKE-specific webhooks, EKS IAM
  for ServiceAccounts). Those need a real cloud cluster — see
  [`k3d`](../k3d/) for the smaller-footprint local equivalent or
  use a per-PR managed cluster.

## Alternatives in this catalog

- [`kind`](../kind/) — Kubernetes-in-Docker, faster to boot,
  smaller footprint, no hypervisor option. Pick `kind` when
  you live inside Docker and want the lightest possible local
  cluster; pick `minikube` when you want hypervisor isolation,
  multiple drivers, or the broader addons catalog (ingress,
  registry, metrics-server, gvisor, …).
- [`k3d`](../k3d/) — k3s-in-Docker. Even smaller, k3s flavor
  rather than upstream kubeadm. Pick `k3d` when you target
  k3s in production (edge / IoT) or need the lowest memory
  footprint.
- [`talosctl`](../talosctl/) — manages Talos Linux clusters
  (api-driven, immutable). Different niche: production-shaped
  clusters, not laptop dev.
- [`k0sctl`](../k0sctl/) — k0s cluster bootstrapper. Production
  installer, not a dev-loop tool.
- [`tilt`](../tilt/) — orchestrates dev workflows *on top of* a
  cluster (rebuild / sync / forward). Pair `minikube` (the
  cluster) with `tilt` (the loop).
- [`devspace`](../devspace/) — same pairing as tilt, different
  flavor. Cluster from minikube, dev loop from devspace.
- [`vcluster`](../vcluster/) — virtual clusters inside a real
  cluster. The "shared dev cluster, isolated namespaces" answer
  that minikube does not target.

## AI-native angle

minikube is not an LLM tool, but it is the substrate every
agent-driven Kubernetes workflow lands on for verification:

- **Real cluster, scriptable lifecycle.** Agents
  ([`aider`](../aider/), [`opencode`](../opencode/),
  [`claude-code`](../claude-code/), [`codex`](../codex/)) can
  `minikube start && kubectl apply -f && minikube delete` as
  a verification shell, getting real API-server feedback on
  manifests they generate instead of static linting alone.
- **Profile isolation for parallel agents.** `minikube start
  -p agent-job-42` creates an isolated cluster profile per
  task; multiple agent runs do not collide on a shared
  kubeconfig context.
- **Deterministic K8s version pinning.** Agents writing
  manifests for a specific cluster version can boot the
  matching minikube and verify locally before shipping a PR.
- **Image-load shortcut.** `minikube image load my/image:sha`
  skips a registry push, which matters when an agent rebuilds
  a container 20 times during one session.

## Caveats

- **Resource hungry.** A multi-node cluster with addons can
  pin 4–6 GB RAM and several CPU cores. Laptop fans will
  spin; CI runners need at least medium-tier hardware.
- **Driver matters.** `docker` is the default and easiest;
  hypervisor drivers (HyperKit / KVM2 / QEMU) provide stronger
  isolation but are more fragile across host upgrades. Pin
  the driver in CI (`--driver=docker --container-runtime=containerd`).
- **Not production parity.** Networking (especially
  LoadBalancer / Ingress) behaves differently on a single
  laptop than on a real cloud cluster. Test cloud-specific
  behavior on a real cloud cluster before release.
- **`minikube tunnel` requires sudo / admin** to bind
  privileged ports on the host. Forget that and Services
  appear `<pending>`.
- **State lives in `~/.minikube/`.** Disk usage grows with
  profiles and cached images; `minikube delete --all
  --purge` is the periodic cleanup.

## Concrete example

```sh
# Boot a 3-node cluster on Kubernetes 1.34, with ingress + registry.
minikube start \
  --profile=dev \
  --driver=docker \
  --nodes=3 \
  --kubernetes-version=v1.34.0 \
  --container-runtime=containerd \
  --cpus=4 --memory=6g

minikube -p dev addons enable ingress
minikube -p dev addons enable registry
minikube -p dev addons enable metrics-server

# Build locally, load straight into the cluster, deploy.
docker build -t myapp:dev .
minikube -p dev image load myapp:dev

kubectl apply -f deploy.yaml
kubectl wait --for=condition=available deploy/myapp --timeout=120s
kubectl port-forward svc/myapp 8080:80 &

# Iterate, then nuke when done.
minikube -p dev delete
```
