# multipass

> **One-command Ubuntu VMs for developers, on macOS / Windows /
> Linux.** Canonical's lightweight VM manager that spins up a
> cloud-init-bootstrapped Ubuntu instance in ~20 seconds with
> the same `cloud-config` you would use on EC2 / Azure — local
> hypervisor (QEMU on macOS / Linux, Hyper-V on Windows) but
> the workflow is `multipass launch / shell / exec / mount /
> stop / delete`, not "boot a hypervisor GUI". Pinned to
> **v1.16.2**
> ([LICENSE](https://github.com/canonical/multipass/blob/main/LICENSE),
> GPL-3.0).

Source: <https://github.com/canonical/multipass>

## TL;DR

`multipass launch --name dev --cpus 4 --memory 8G --disk 40G
24.04` returns a fresh Ubuntu 24.04 VM in well under a minute,
with cloud-init applied, an SSH key injected, and a
host-to-guest mount available via `multipass mount $PWD
dev:/work`. `multipass shell dev` drops you into a real `bash`
on a real kernel — useful for testing systemd units, kernel
modules, iptables / nftables rules, snap packages, and Linux
containers in a way Docker-on-Mac / WSL2 cannot reproduce.
Ships its own driver layer so the same `multipass` CLI works
unchanged on Apple Silicon (QEMU + HVF), Intel macOS, Windows
(Hyper-V), and Linux (KVM).

## Install

```bash
# macOS
brew install --cask multipass

# Linux (snap)
sudo snap install multipass

# Windows
winget install Canonical.Multipass
```

## Example

```bash
# Launch a 4-core / 8 GB / 40 GB Ubuntu 24.04 VM
multipass launch --name dev --cpus 4 --memory 8G --disk 40G 24.04

# Apply cloud-init at boot (same syntax as EC2 user-data)
cat <<'EOF' > cloud.yaml
#cloud-config
packages: [build-essential, git, jq]
runcmd:
  - [ systemctl, enable, --now, ssh ]
EOF
multipass launch --name ci --cloud-init cloud.yaml 24.04

# Mount host code into the VM
multipass mount "$PWD" dev:/work

# Run a command without an interactive shell
multipass exec dev -- bash -lc 'cd /work && make test'

# Snapshot, restore, delete
multipass snapshot dev --name pre-upgrade
multipass restore dev.pre-upgrade
multipass delete dev && multipass purge
```

## When to use

- You need a real Linux kernel on a Mac / Windows host to test
  systemd units, kernel modules, nftables rules, AppArmor
  policies, or snap packaging — Docker-on-Mac shares the host
  kernel via a shim and can't reproduce these.
- You want one VM CLI that behaves the same on Apple Silicon,
  Intel macOS, Windows (Hyper-V), and Linux (KVM) without
  learning four different hypervisor frontends.
- You want to dry-run a `cloud-init` user-data script locally
  before shipping it to EC2 / Azure / Hetzner.
- You want a disposable Ubuntu sandbox for a workshop / demo /
  live-coding session that resets to a clean snapshot.

## When NOT to use

- You only need a containerised Linux for building / running
  applications — Docker / Podman / `lima` / `colima` / `orbstack`
  are lighter and faster.
- You need non-Ubuntu guests (RHEL, Alpine, FreeBSD, Windows) —
  `multipass` is Ubuntu-only by design; reach for `lima`,
  `vagrant`, `tart`, or raw QEMU / virt-manager.
- You need a multi-VM cluster with declarative networking
  (overlays, BGP, VRFs) — use `vagrant` + libvirt, `terraform` +
  a hypervisor provider, or move to a real Kubernetes / OpenStack
  setup.
