# k3d

- **Repo:** https://github.com/k3d-io/k3d
- **Version:** v5.8.3
- **License:** [LICENSE](https://github.com/k3d-io/k3d/blob/main/LICENSE) (MIT)
- **Category:** Container / Kubernetes (local clusters)

## What it is

k3d wraps Rancher's k3s lightweight Kubernetes distribution in Docker
containers, letting you spin up multi-node clusters on a laptop in seconds with
a single `k3d cluster create`. Each node is a container, and the binary handles
load-balancer wiring, image imports, registry creation, and kubeconfig merging
for you.

## Why it's interesting

- **Multi-node clusters in seconds** without a VM — `k3d cluster create dev
  --servers 1 --agents 3 --port 8080:80@loadbalancer` is one command vs. the
  usual minikube/kind dance.
- **Built-in `k3d image import` and `k3d registry create`** make the
  build → load → deploy loop a single subcommand instead of a private-registry
  setup project.
- **Pairs with kind/minikube as the "fastest startup" option** because k3s
  trims etcd in favour of an embedded SQLite/Kine backend, so cold-start cluster
  boot is sub-30s on common hardware.
