# lima

> **Linux VMs on macOS / Linux with automatic file sharing,
> port forwarding, and containerd preinstalled.** A `limactl`
> CLI that boots a lightweight QEMU or Apple Virtualization
> Framework guest from a YAML template, mounts your `$HOME`
> read-write into the VM, forwards ports automatically, and
> exposes `lima <cmd>` as a transparent remote-shell prefix.
> Pinned to **v2.1.1** ([LICENSE](https://github.com/lima-vm/lima/blob/v2.1.1/LICENSE),
> Apache-2.0).

Source: <https://github.com/lima-vm/lima>

## TL;DR

`lima` started as "containerd-on-macOS the easy way" and grew
into a general-purpose Linux-VM runner. `limactl create
template://default` writes a `default.yaml` describing CPU,
memory, disk, image (Ubuntu / AlmaLinux / Fedora / Debian /
Arch / openSUSE templates ship in-tree), mounted host paths,
and provisioning scripts; `limactl start default` boots it and
runs cloud-init. Once running, `lima <cmd>` is the same as
`limactl shell default <cmd>` — the host `$HOME` is mounted at
the same path inside the guest, so `lima go build ./...` from
your project directory Just Works. VM type is selectable
(`vmType: vz` for Apple's hypervisor on Apple Silicon, `qemu`
for portable, `wsl2` on Windows). Templates with `containerd`
or `docker` preinstalled make it the standard backend for
[`colima`](../colima/) and Homebrew's `nerdctl` formula.

## Install

```bash
# Homebrew (macOS / Linux)
brew install lima

# MacPorts
sudo port install lima

# Direct binary (macOS arm64)
curl -fsSL -o lima.tar.gz \
  https://github.com/lima-vm/lima/releases/download/v2.1.1/lima-2.1.1-Darwin-arm64.tar.gz
sudo tar Cxzvf /usr/local lima.tar.gz
```

## Example

```bash
# Boot the default Ubuntu VM with containerd + nerdctl preinstalled
limactl start template://default

# Run a command in the VM (host $HOME is mounted)
lima uname -a
lima nerdctl run -d --name web -p 8080:80 nginx:alpine
curl http://localhost:8080      # port-forwarded to the host

# Boot a Fedora VM with 4 CPUs / 8 GiB / 50 GiB disk
limactl create --name fed --cpus 4 --memory 8 --disk 50 template://fedora
limactl start fed
limactl shell fed dnf install -y go

# Snapshot, pause, and clean up
limactl snapshot create default --tag clean
limactl stop default && limactl delete default
```

## When to use

- You are on macOS / Apple Silicon and need a real Linux kernel
  for development (eBPF, KVM-only tooling, distro-specific
  packaging) without paying for Docker Desktop or running a
  full VirtualBox install.
- You want one YAML file under version control that anyone on
  the team can `limactl start` to get the identical dev VM.
- You want containerd / nerdctl on macOS — `brew install
  nerdctl` already uses Lima under the hood.

## When NOT to use

- You only need a container runtime, not a full Linux VM —
  `colima start` (which uses Lima internally) is the
  higher-level convenience; `podman machine init` is another
  option.
- You need GPU passthrough or DirectX — Lima targets headless
  Linux dev workloads, not gaming / ML training (use UTM /
  Parallels / native Linux).
- You are already on a Linux host — Lima still works (KVM
  backend) but adds a VM where containers / namespaces would
  do; use plain `nerdctl` / `podman` directly.
