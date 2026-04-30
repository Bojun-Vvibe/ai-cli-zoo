# cilium-cli

> **Install, upgrade, and troubleshoot Cilium on Kubernetes.** A
> single Go binary that wraps the lifecycle of the Cilium CNI
> (eBPF-based networking, network policy, and service mesh) so
> the same `cilium install` / `cilium status` / `cilium connectivity test`
> commands work against kind, k3d, EKS, GKE, AKS, and bare-metal
> clusters. Pinned to **v0.19.2**
> (per upstream releases page, verified 2026-05-01)
> ([LICENSE](https://github.com/cilium/cilium-cli/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/cilium/cilium-cli>

## TL;DR

`cilium-cli` is the operator-facing front door to a Cilium
deployment. `cilium install` provisions the agent + operator
DaemonSet/Deployment with sane per-cloud defaults (correct CNI
chaining for EKS, IPAM mode for GKE, kube-proxy replacement
toggles for kind), `cilium status --wait` blocks until the data
plane is healthy, and `cilium connectivity test` runs a
~50-test pod-to-pod / pod-to-service / pod-to-world matrix that
catches MTU mismatches, dropped policy decisions, and broken
egress *before* a real workload trips on them. Hubble
sub-commands (`cilium hubble enable`, `cilium hubble port-forward`)
wire up the observability layer in two commands.

## Install

```bash
# Homebrew (macOS / Linux)
brew install cilium-cli

# Direct download (Linux amd64 example)
curl -L --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/v0.19.2/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin

# Docker
docker run --rm -v "$HOME/.kube/config":/root/.kube/config \
  quay.io/cilium/cilium-cli-ci:v0.19.2 status
```

## Example

```bash
# Install Cilium with kube-proxy replacement on a kind cluster
cilium install --version 1.16.4 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=kind-control-plane \
  --set k8sServicePort=6443

# Wait for the data plane and run the connectivity matrix
cilium status --wait
cilium connectivity test --test '!host-firewall'

# Enable Hubble + open the relay locally
cilium hubble enable --ui
cilium hubble port-forward &
hubble observe --follow
```

## When to use

- You are bootstrapping Cilium on a new cluster and want the
  per-cloud defaults figured out for you instead of hand-rolling
  Helm values.
- You suspect a CNI / policy / MTU issue in a running cluster
  and want a deterministic, repeatable connectivity matrix to
  bisect against.
- You want the Hubble observability stack (flow logs, service
  map, DNS visibility) wired in without learning the underlying
  CRDs first.

## When NOT to use

- You manage Cilium via Helm + GitOps already — the CLI's
  `install` path competes with your `HelmRelease`. Use the CLI
  only for `status` / `connectivity test` / `hubble` sub-commands
  in that case.
- You want fine-grained control over every Helm value — the CLI
  exposes the common ones and forwards the rest via `--set`,
  but the Helm chart is still the source of truth for advanced
  tuning.
- You need to manage a non-Cilium CNI (Calico, Flannel,
  Antrea) — this CLI is Cilium-specific by design.
