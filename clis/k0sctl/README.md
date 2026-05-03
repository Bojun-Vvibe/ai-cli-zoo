# k0sctl

> **Declarative bootstrapper for k0s Kubernetes clusters over
> SSH.** A single Go binary that reads a YAML cluster spec
> (controllers, workers, host addresses, SSH keys, k0s version)
> and idempotently installs, upgrades, resets, or backs up the
> cluster — no agent on the hosts, no Ansible, no extra control
> plane. Pinned to **v0.30.0**
> ([LICENSE](https://github.com/k0sproject/k0sctl/blob/main/LICENSE),
> Apache-2.0; docs under CC-BY-SA-4.0).

Source: <https://github.com/k0sproject/k0sctl>

## TL;DR

`k0sctl apply --config k0sctl.yaml` SSHes into every host in
the spec, downloads the pinned `k0s` binary, configures roles
(`controller`, `controller+worker`, `worker`), starts the
systemd service, joins workers to the control plane, and writes
back a working `kubeconfig` you can hand to `kubectl`. Re-run
to upgrade k0s in place (rolling, controllers first), reset
(`k0sctl reset`) to nuke the cluster cleanly, or `k0sctl backup`
/ `restore` to snapshot etcd / kine state. Hosts are addressed
by SSH (`ssh:`), local exec (`localhost:`), or WinRM for Windows
workers.

## Install

```bash
# Homebrew (macOS / Linux)
brew install k0sproject/tap/k0sctl

# Direct binary (pinned)
curl -sSLO https://github.com/k0sproject/k0sctl/releases/download/v0.30.0/k0sctl-linux-amd64
chmod +x k0sctl-linux-amd64 && sudo mv k0sctl-linux-amd64 /usr/local/bin/k0sctl

# Container
docker run --rm -v "$PWD:/work" -w /work \
  ghcr.io/k0sproject/k0sctl:v0.30.0 apply --config k0sctl.yaml
```

## Example

```bash
# Generate a starter spec from a list of hosts and edit it
k0sctl init --k0s controller worker@10.0.0.10 worker@10.0.0.11 > k0sctl.yaml

# Apply (install or upgrade in place)
k0sctl apply --config k0sctl.yaml

# Pull a kubeconfig for the cluster
k0sctl kubeconfig --config k0sctl.yaml > kubeconfig
KUBECONFIG=$PWD/kubeconfig kubectl get nodes

# Backup etcd / kine state before a risky upgrade
k0sctl backup --config k0sctl.yaml
```

## When to use

- You run k0s on bare metal, edge nodes, or VMs and want a
  reproducible YAML for cluster lifecycle instead of hand-
  rolled scripts.
- You need rolling, version-pinned upgrades of the control
  plane and workers from one command.
- You want SSH-only bootstrapping with no agent left behind on
  the hosts.

## When NOT to use

- You are on a managed k8s service (EKS / GKE / AKS) — k0sctl
  has nothing to do; use the cloud provider's tools.
- You want full kubeadm semantics or a different distribution
  — k0sctl is k0s-specific.
- The cluster is single-node throwaway dev — `k0s install` or
  `kind` / `k3d` are lighter.

## Niche / tags

`k8s-helper` · `cluster-bootstrap` · `k0s` · `ssh` · `go`
