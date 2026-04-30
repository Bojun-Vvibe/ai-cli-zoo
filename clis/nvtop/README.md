# nvtop

- **Repo:** https://github.com/Syllo/nvtop
- **Latest version:** 3.3.2 (2026-02-08)
- **HEAD on `master`:** `095d91c`
- **License:** GPL-3.0 — [LICENSE](https://github.com/Syllo/nvtop/blob/master/LICENSE)
- **Category:** sysadmin / GPU monitor

A **GPU-aware `htop`** for Linux and FreeBSD. `nvtop` is a single-binary
ncurses dashboard that shows real-time GPU utilization, per-process VRAM
consumption, temperature, fan speed, power draw, encoder/decoder usage,
and PCIe bandwidth across **multiple GPU vendors in one view**: NVIDIA
(via NVML), AMD (via libdrm + amdgpu sysfs), Intel (i915 / Xe), Adreno
(Qualcomm), Apple (M-series via IOKit on macOS — partial), Ascend
(Huawei), and Tenstorrent. The killer feature for ML workflows is the
per-process attribution: it does not just say "VRAM is 80% full", it
tells you which Python PID is holding which 6 GiB.

## Install

```bash
# Debian / Ubuntu
sudo apt install nvtop

# Fedora / RHEL
sudo dnf install nvtop

# Arch
sudo pacman -S nvtop

# Homebrew (Linux only — no GPU APIs on macOS Homebrew formula)
brew install nvtop

# from source (CMake + ncurses + libdrm + the relevant vendor SDK)
git clone https://github.com/Syllo/nvtop && cd nvtop
mkdir build && cd build
cmake .. -DNVIDIA_SUPPORT=ON -DAMDGPU_SUPPORT=ON -DINTEL_SUPPORT=ON
make && sudo make install

# verify
nvtop --version    # nvtop version 3.3.2
```

## Sample usage

```bash
# launch the live dashboard (auto-detects all GPUs)
nvtop

# show only NVIDIA GPUs (vendor filter)
nvtop -s NVIDIA

# 100 ms refresh, useful when chasing a transient kernel
nvtop -d 1

# limit which GPUs are shown (comma-separated indices)
nvtop -i 0,2

# headless / one-shot snapshot for a log
nvtop -p > /tmp/gpu-snapshot.txt   # not a stable text contract; for humans

# inside the TUI:
#   F2  setup screen (toggle columns, sort key, vendor filter)
#   F6  sort by GPU%, MEM%, PID, USER, COMMAND
#   F9  send SIGTERM to the highlighted process
#   F10 quit
```

## Niche it fills

The **"multi-vendor `htop` for GPUs"** slot. `nvidia-smi` is the
canonical NVIDIA tool but only sees NVIDIA cards and ships a
verbose, scriptable, *non-interactive* table. `rocm-smi` is the AMD
counterpart with the same scope limit. `nvtop` is the tool you keep
on a workstation that has both an NVIDIA card and an AMD card (or a
laptop with an Intel iGPU + a discrete dGPU) when you want one
ncurses pane that shows everything, sorted by VRAM, with a kill key.

## Why it matters

- **Per-process VRAM attribution across vendors.** `nvidia-smi
  --query-compute-apps=pid,used_memory` does this for NVIDIA only;
  `nvtop` unifies the view across NVIDIA / AMD / Intel using
  vendor-specific syscalls under the hood.
- **Encoder / decoder utilization columns.** Useful for video
  workloads (ffmpeg, OBS) where the SM% / shader-busy% number is
  zero but NVENC / VAAPI is saturated; classic monitors miss this.
- **Headless-friendly build.** Compile with only the vendor support
  you need (CMake `-DNVIDIA_SUPPORT=OFF -DAMDGPU_SUPPORT=ON`) for
  ROCm-only servers without dragging in NVML.
- **No daemon, no root.** Reads sysfs and vendor APIs directly with
  the running user's permissions; install it on a shared GPU box
  without filing a ticket.

## Caveats

- **Output is for humans.** No JSON / CSV mode; use `nvidia-smi
  --query-gpu=...` or `rocm-smi --json` when you want machine-
  parseable data, then keep `nvtop` for interactive use.
- **GPL-3.0 — not LGPL or MIT.** Linking or bundling into a
  proprietary product needs care; running it on your own boxes is
  fine.
- **macOS support is partial.** Apple Silicon shows basic
  utilization via IOKit but lacks the per-process attribution that
  Linux gets; use `asitop` / `mactop` as the macOS-native counterpart.
- **Vendor SDK headers required at build time** if you compile from
  source — the distro packages already bundle the right combinations.
- **Power / temperature columns depend on driver version.** Old
  NVIDIA drivers (< R515), out-of-tree AMD modules, and Intel
  Arc-on-old-kernels may show `N/A` for some columns.
