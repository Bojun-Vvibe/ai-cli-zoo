# talosctl

- **Repo:** https://github.com/siderolabs/talos
- **Version:** v1.13.0 (2026-04-27)
- **License:** MPL-2.0 ([LICENSE](https://github.com/siderolabs/talos/blob/main/LICENSE))
- **Language:** Go; ships static single-file `talosctl` binaries plus the Talos Linux OS image (kernel + initramfs + squashfs) for amd64 / arm64
- **Install:** `brew install siderolabs/tap/talosctl` · `curl -sL https://talos.dev/install | sh` · pre-built binaries on the [releases page](https://github.com/siderolabs/talos/releases) for macOS / Linux / Windows · Docker `ghcr.io/siderolabs/talosctl:v1.13.0`

## What it does

`talosctl` is the operator-side CLI for **Talos Linux**, an immutable, API-driven Linux distribution purpose-built for running Kubernetes — no shell, no SSH, no package manager, no systemd-style init scripts on the node. The entire host OS is a read-only squashfs (the rootfs is mounted r/o; only `/var` and `/etc/machine-id` are writable), every node-level operation is a gRPC call against the on-host `apid` (port 50000) authenticated by mTLS, and `talosctl` is the only supported way to interact with a node. The command surface mirrors `kubectl` deliberately: `talosctl get members`, `talosctl get services`, `talosctl logs kubelet`, `talosctl dmesg`, `talosctl read /proc/cpuinfo`, `talosctl restart`, `talosctl upgrade --image ghcr.io/siderolabs/installer:v1.13.0`, `talosctl reset --graceful=true --reboot=true`. Cluster bring-up is two commands: `talosctl gen config <cluster-name> https://<vip>:6443` produces three YAML machine configs (`controlplane.yaml`, `worker.yaml`, `talosconfig`), then `talosctl apply-config --insecure --nodes <ip> --file controlplane.yaml` patches a freshly-PXE-booted (or `talosctl cluster create` Docker-driven local) node into a control-plane member, then `talosctl bootstrap --nodes <first-cp>` runs the etcd bootstrap once, then `talosctl kubeconfig` writes a `kubectl`-usable kubeconfig and the cluster is ready. The same CLI handles disk wipe + re-install (`talosctl wipe disk sda`), in-place OS upgrades with automatic rollback (`talosctl upgrade --preserve` keeps `/var`, `--stage` defers to next reboot), Kubernetes version upgrades (`talosctl upgrade-k8s --to v1.34.2` rolls control-plane and kubelets one at a time), etcd member management (`talosctl etcd members`, `talosctl etcd remove-member`, `talosctl etcd snapshot /tmp/etcd.snap`), and cluster-wide diagnostics (`talosctl health`, `talosctl support` builds a redacted bundle for upstream issue triage). A killer subcommand is `talosctl cluster create`, which boots a complete multi-node Talos cluster as Docker containers (or QEMU VMs on Linux with `--provisioner qemu`) on the local laptop in under 60 seconds for development, with the same machine-config schema as production.

## When to pick it / when not to

Pick `talosctl` (and Talos Linux) when running Kubernetes is the **only** workload of the host OS and operational toil from "managing Linux" is the bottleneck — homelab clusters, edge fleets, bare-metal production, air-gapped factories, multi-region appliance deployments. Concrete cases: a homelab where 4 mini-PCs run a 1-CP / 3-worker cluster and you want zero node-level maintenance (no `apt upgrade` saturday, no `systemd` unit drift, no SSH key rotation — just `talosctl upgrade` once a month and the OS image is replaced atomically with rollback on boot failure); an edge fleet where 200 retail-store NUCs all need the same Kubernetes version and machine config, declaratively reconciled, with no possibility of an on-site tech `vi`-ing `/etc/sysctl.conf` and drifting one site; a bare-metal datacenter where Talos's PXE / iPXE boot path means new servers come up with the right config from a Matchbox / Sidero Metal endpoint and join the cluster automatically (`talosctl get machinestatus` shows the live state); a regulated environment that needs to demonstrate "no shell access to the host" as an audit control. Pair with [`flux`](../flux/) or [`argocd`](../argocd/) for the workload-side GitOps after `talosctl bootstrap`; pair with [`cilium-cli`](../cilium-cli/) when Cilium is the chosen CNI (Talos has first-class Cilium docs and a `--config-patch` recipe); pair with [`k9s`](../k9s/) for the in-cluster TUI once kubeconfig exists; pair with [`omnictl`](https://github.com/siderolabs/omni) — Sidero's commercial SaaS / self-hostable management plane — when the fleet outgrows manual `talosctl --nodes` lists.

Skip `talosctl` when the host OS needs to run anything other than Kubernetes — a database directly on `systemd`, a legacy daemon that wants `/etc/init.d`, an SSH-driven Ansible workflow, an LDAP-joined login shell. Talos genuinely has no shell; the design assumption is "all workloads are containers, scheduled by Kubernetes". Skip when a single-node Kubernetes is enough and `k3s` / `k0s` / `microk8s` (which run on a normal Linux you already manage) are sufficient — Talos's value compounds with fleet size and homogeneity. Skip on cloud-managed Kubernetes (EKS / GKE / AKS) where the cloud already manages the node OS; Talos's niche is where you own the metal. Skip when the team's operational muscle memory is built on `ssh + journalctl + tail -f`; Talos's gRPC-only model is a real adjustment and the learning investment is non-trivial for a one-off cluster.

## Vs already cataloged

- **Vs [`k0s`](../k0s/) / [`k3s`](../k3s/) / [`microk8s`](../microk8s/):** different layer. Those are *Kubernetes distributions* that run on top of an existing Linux you maintain (Ubuntu / Debian / RHEL). Talos is a *Linux distribution* whose only purpose is hosting Kubernetes — there is no underlying OS for you to maintain. Use `k0s` / `k3s` when you want lightweight Kubernetes on hosts you already operate; use Talos when you want to stop operating hosts.
- **Vs [`kubeadm`](../kubeadm/):** complementary in goal, opposite in philosophy. `kubeadm` bootstraps a Kubernetes control plane on a Linux you brought; `talosctl` boots both the OS *and* the Kubernetes control plane from a single declarative machine config. Talos clusters are not `kubeadm` clusters — etcd, the kubelet, the static-pod manifests are all managed by the on-host Talos `machined`, not `kubeadm`.
- **Vs [`fcct`](https://coreos.github.io/butane/) / Flatcar / Bottlerocket:** same spirit (immutable Linux for containers), different surface. Flatcar / Fedora CoreOS still have a shell, `systemd`, and SSH; Bottlerocket has an `apiclient` but no full-featured operator CLI. Talos is the most extreme on the "no shell, API-only" axis and `talosctl` is the most complete operator surface in the immutable-Linux niche.
- **Vs [`kubectl`](../kubectl/):** orthogonal. `kubectl` operates on the Kubernetes API (Deployments, Pods, Services); `talosctl` operates on the Talos node API (the OS, etcd, kubelet logs, disks, reboot). Both are needed in a Talos cluster — `talosctl kubeconfig` produces the kubeconfig `kubectl` then uses.
- **Vs [`packer`](../packer/) / [`bottlerocket`](https://github.com/bottlerocket-os/bottlerocket):** Talos ships a pre-built OS image (you do not build your own); Packer-with-CoreOS / Bottlerocket workflows assume image build. The Talos answer to "I need a custom kernel module" is the [Image Factory](https://factory.talos.dev) — a hosted (or self-hostable) image-builder that produces signed Talos images with the chosen system extensions baked in.
- **Vs cloud-managed Kubernetes (EKS / GKE / AKS):** different problem. Managed services hide the node OS by *running it for you*; Talos hides the node OS by *making it not require management*. On bare metal or homelab where there is no managed option, Talos is the closest experience.

## Caveats

- **No SSH, no shell, ever.** This is the design, not a missing feature. Debugging a misbehaving kubelet means `talosctl logs kubelet`, `talosctl dmesg`, `talosctl read /sys/...`, `talosctl service kubelet restart`. There is no `bash` to `kubectl exec` into on the host. Plan operational runbooks around the gRPC API or do not adopt Talos.
- **Machine config schema evolves between minor versions.** A v1.12 machine config is not always valid for v1.13 — read the [upgrade notes](https://www.talos.dev/v1.13/talos-guides/upgrading-talos/) before each upgrade, run `talosctl upgrade --dry-run` first, and keep the rendered configs in git so the diff is reviewable.
- **mTLS-only access; the `talosconfig` is the keys to the cluster.** Treat `~/.talos/config` like a kubeconfig + an SSH private key combined. Loss of the `talosconfig` plus loss of the bootstrap PKI (`secrets.yaml` from `talosctl gen secrets`) means losing the ability to manage the cluster — back both up out-of-band.
- **`talosctl reset` is destructive and fast.** `talosctl reset --graceful=false --reboot=true --nodes 10.0.0.5` wipes the system disk and reboots in seconds with no confirmation prompt past the flag. Always `--dry-run` first against the wrong node-list, especially when `--nodes` is taken from a script.
- **In-place upgrade with `--preserve` keeps `/var` but not `/etc`.** `/etc/machine-id` is preserved; user-added files under `/etc` (there should not be any — Talos discourages it) are not. The upgrade installer image must match the architecture of the node; mismatches are caught early but `--image` typos are silent until reboot.
- **Networking on bare metal needs forethought.** Talos's `controlplane.yaml` declares VIP / bond / VLAN / bridge interfaces; getting the DHCP / static-IP / `vip` setup right pre-bootstrap matters because there is no shell to `ip link` your way out of a misconfiguration. Use `talosctl cluster create` locally first to validate the config shape.
- MPL-2.0 ([LICENSE](https://github.com/siderolabs/talos/blob/main/LICENSE)) — file-level copyleft (modifications to MPL-2.0 files must be shared upstream); does not infect non-MPL workloads running on the cluster. Sidero Labs' commercial Omni management plane is separately licensed and optional.

## Example invocations

```bash
# Install
brew install siderolabs/tap/talosctl
# or
curl -sL https://talos.dev/install | sh

# Local 3-node Docker-based cluster for development (60s)
talosctl cluster create --name dev --workers 2 --controlplanes 1
talosctl --talosconfig ~/.talos/clusters/dev/talosconfig kubeconfig .
export KUBECONFIG=$PWD/kubeconfig
kubectl get nodes

# Production: generate machine configs against a VIP
talosctl gen config prod-cluster https://10.0.0.10:6443 \
  --output-dir ./prod-config

# Apply a freshly-booted node into the control plane (insecure = pre-PKI)
talosctl apply-config --insecure --nodes 10.0.0.11 \
  --file ./prod-config/controlplane.yaml

# Bootstrap etcd (one time, on the first control-plane only)
talosctl --talosconfig ./prod-config/talosconfig \
  --nodes 10.0.0.11 bootstrap

# Pull kubeconfig
talosctl --talosconfig ./prod-config/talosconfig \
  --nodes 10.0.0.11 kubeconfig ./prod.kubeconfig

# Read live state
talosctl --nodes 10.0.0.11,10.0.0.12,10.0.0.13 get members
talosctl --nodes 10.0.0.11 logs kubelet
talosctl --nodes 10.0.0.11 dmesg
talosctl --nodes 10.0.0.11 dashboard   # ncurses-style live TUI

# In-place OS upgrade (rollback-on-failure built in)
talosctl --nodes 10.0.0.11 upgrade \
  --image ghcr.io/siderolabs/installer:v1.13.0 --preserve

# Kubernetes upgrade (rolls CP + kubelets one node at a time)
talosctl --nodes 10.0.0.11 upgrade-k8s --to v1.34.2

# etcd snapshot for DR
talosctl --nodes 10.0.0.11 etcd snapshot /tmp/etcd-$(date +%F).snap

# Build a redacted support bundle for upstream issue triage
talosctl --nodes 10.0.0.11,10.0.0.12,10.0.0.13 support \
  -O support-$(date +%F).zip
```
