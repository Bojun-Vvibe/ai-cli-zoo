# kind

- **Repo:** https://github.com/kubernetes-sigs/kind
- **Version:** v0.31.0
- **License:** [LICENSE](https://github.com/kubernetes-sigs/kind/blob/main/LICENSE) (Apache-2.0)
- **Category:** Container / Kubernetes (local clusters)

## What it is

kind ("Kubernetes IN Docker") is the Kubernetes-SIGs project for running
local clusters where every node is a Docker (or Podman) container running a
real `kubelet`. It is the same tool the upstream Kubernetes project uses to
run conformance and e2e tests in CI, so the cluster you boot on a laptop is
configured to match the way upstream actually exercises Kubernetes — kubeadm
bootstrap, real `etcd`, real control-plane components, configurable
`kubeadm` patches, and pluggable CNI.

## Why it's interesting

- **Closest-to-upstream local cluster.** Unlike k3s-based options that swap
  `etcd` for SQLite/Kine and trim features, kind boots an unmodified
  upstream control plane, which is why it is the reference rig for
  Kubernetes' own SIG test suites.
- **Multi-node + HA control plane in one YAML.** A `kind: Cluster` config
  with `nodes:` of role `control-plane` / `worker` gives you a 3-master HA
  cluster on a laptop — useful for testing leader election, taints,
  topology spread, or PodDisruptionBudget behaviour.
- **`kind load docker-image` / `kind load image-archive`** import local
  images directly into all node containers without a registry round-trip,
  which is the fastest inner-loop for "build → deploy → debug" on
  Kubernetes manifests, Helm charts, or operators.
- **First-class CI story.** Single static binary, deterministic node image
  digest pinning (`kindest/node:v1.34.x@sha256:...`), and an official
  `kind-action` for GitHub Actions make it the default choice for
  per-PR ephemeral Kubernetes integration tests.

## Install

```bash
# Homebrew
brew install kind

# Go install
go install sigs.k8s.io/kind@v0.31.0

# Static binary (Linux/macOS/Windows)
# https://github.com/kubernetes-sigs/kind/releases

kind version    # kind v0.31.0 ...
```

## Examples

```bash
# 1-node cluster, default Kubernetes version
kind create cluster --name dev

# 3-node cluster (1 control-plane + 2 workers) pinned to a Kubernetes minor
cat <<'EOF' | kind create cluster --name multi --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

# load a locally-built image into the cluster (no registry needed)
docker build -t myapp:dev .
kind load docker-image myapp:dev --name multi

# tear it all down
kind delete cluster --name multi
```

## Use when

- You want a local Kubernetes that behaves *exactly* like upstream — you are
  developing operators, admission webhooks, CRDs, or anything that touches
  the API machinery and you cannot tolerate the small behavioural deltas of
  k3s or minikube's various drivers.
- Your CI runs in GitHub Actions / GitLab CI / Buildkite and you want
  per-job ephemeral Kubernetes clusters with image preloading and no VM.
- You need to test multi-node concerns (HA control plane, node failure,
  topology spread, taints/tolerations) without paying for cloud nodes.

## Use something else when

- You want sub-30-second cold-start on weak hardware: `k3d` boots faster
  because k3s strips etcd in favour of Kine/SQLite. See [`k3d`](../k3d/).
- You need a desktop GUI, addons UI, or built-in dashboard out of the
  box — `minikube` is friendlier for first-time learners.
- You want to develop *against* Kubernetes APIs without running a real
  control plane at all — `envtest` (controller-runtime) gives you just
  `etcd` + `kube-apiserver` for unit tests in milliseconds.

## Alternatives

- [`k3d`](../k3d/) — k3s-in-Docker, faster cold-start, lighter footprint.
- `minikube` — VM-based or Docker-based, friendlier UX, more drivers.
- `microk8s` — snap-packaged single-node for Ubuntu hosts.
- `colima` — macOS-focused container runtime that can also surface a
  Kubernetes cluster (k3s under the hood).
