# nvitop

> **An interactive NVIDIA-GPU process viewer that does
> for `nvidia-smi` what `htop` did for `top`** —
> per-GPU utilization / memory / power / temperature in
> color-coded bars at the top, a per-process table
> below (PID, user, GPU mem, GPU-util, CPU%, RSS,
> command), all sortable / filterable / killable from
> the keyboard, plus tree / chart / monitor modes
> (`nvitop`, `nvitop -m auto`, `nvitop -m full`) and a
> drop-in Python API (`from nvitop import Device,
> ResourceMetricCollector`) when you want to *script*
> against the same data your eyes are reading. Pinned
> to **v1.6.2** ([LICENSE](https://github.com/XuehaiPan/nvitop/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/XuehaiPan/nvitop>

## TL;DR

`nvidia-smi` is the universal answer to "what is my
GPU doing", but its output is a static snapshot per
invocation, the column layout reflows with terminal
width unpredictably, killing a runaway PyTorch process
requires a separate `kill` window, and tabular log
parsing is fragile. `nvitop` is the focused modern
alternative: a Textual-style live TUI built on
`nvidia-ml-py` (NVML) that refreshes ~1 Hz, surfaces
the same metrics in color-coded bars, lets you sort
the process list by any column, and binds `K` to send
SIGKILL to the focused PID after a confirmation prompt.
The same library is importable as a Python module for
training-loop instrumentation: `with
ResourceMetricCollector(...) as collector:` records
per-step GPU usage you can dump to TensorBoard.

## Install

```bash
# pipx (recommended — keeps deps isolated from your project envs)
pipx install nvitop

# pip
pip install --upgrade nvitop

# conda-forge
conda install -c conda-forge nvitop

# Poetry / PDM project add
poetry add nvitop
pdm add nvitop

# Specific version
pip install --upgrade 'nvitop==1.6.2'

# verify
nvitop --version    # nvitop 1.6.2
```

`nvitop` requires a host with NVIDIA drivers installed
(it does not ship CUDA itself; it just talks to the
already-loaded kernel module via NVML). Works from
inside containers as long as `--gpus all` (or the
NVIDIA container runtime) makes `/dev/nvidia*` visible.

## Use it for

```bash
# The headline use: live, sortable GPU + process view
nvitop

# Inside the TUI:
# Tab          – cycle focus (devices / processes / commands)
# Up / Down    – move within the focused pane
# Enter        – expand a process to show full command line
# K            – kill the focused PID (asks for signal: TERM / KILL / INT)
# T            – terminate (SIGTERM)
# Space        – pause auto-refresh
# +/-          – change refresh interval
# /            – filter processes by name / user
# s            – sort by column (cycles GPU-util / mem / CPU / time)
# q            – quit

# Monitor modes (output shape, not "kill-this-process" interactivity):
nvitop -m auto    # adaptive: full when terminal is large, compact otherwise
nvitop -m full    # always full (every device + every column)
nvitop -m compact # always compact (one line per device)

# One-shot text snapshot for piping / logging
nvitop --once             # static dump, no TUI
nvitop --once -U          # also include per-user totals

# Filter to specific GPUs / processes
nvitop --only 0,2         # only devices 0 and 2
nvitop --only-visible     # respect $CUDA_VISIBLE_DEVICES (great inside slurm jobs)
nvitop --filter "pytorch" # only processes whose command contains "pytorch"

# Tree-mode (process hierarchy across GPUs)
nvitop --tree
```

The Python API is the second half of the product:

```python
from nvitop import Device, ResourceMetricCollector, NA

# Snapshot the whole node
for device in Device.all():
    print(device.index, device.name(), device.memory_used_human(), device.gpu_utilization())

# Instrument a training loop — record per-step GPU metrics
with ResourceMetricCollector(Device.cuda.all()) as collector:
    for step in range(num_steps):
        train_one_step()
        metrics = collector.collect()      # dict suitable for TensorBoard / W&B / MLflow
        writer.add_scalars("gpu", metrics, step)
```

The `ResourceMetricCollector` integrates cleanly with
TensorBoard / Weights & Biases / MLflow without
shelling out to `nvidia-smi` — same NVML calls,
in-process.

## Why include it in a CLI catalog

1. **It is the focused modern answer to "watch my
   GPU".** Every ML / DL / inference workflow needs
   this answer eventually: which job is hogging memory,
   which model is starving for compute, why did
   `cuda.OutOfMemoryError` fire when I thought 24 GB
   was plenty. `nvidia-smi -l 1` works, but the
   reflowing output and lack of process-row navigation
   make it tiring. `nvitop` is one keystroke
   (`/pytorch`) away from "show me only the PyTorch
   processes" and one keystroke (`K`) away from
   killing the leaker.
2. **The Python API closes the loop.** Most other
   GPU-watching CLIs (`gpustat`, `nvtop`) are
   monitoring-only. `nvitop` ships the *same*
   underlying `Device` class as a public Python API,
   so the metric you saw flicker in the TUI is the
   metric your training script can log per step. This
   is unique in the catalog.
3. **Container- and slurm-aware.** Respects
   `$CUDA_VISIBLE_DEVICES`, so when a SLURM job
   constrains you to GPU 3, `nvitop --only-visible`
   shows you exactly what the job sees — no manual
   indexing, no risk of poking another tenant's
   process. Inside Docker / Podman with the NVIDIA
   container runtime, `nvitop` works without further
   config.

For an LLM-CLI workflow, `nvitop` is the substrate for
"my local Llama 3.1 / Qwen / DeepSeek inference is
slow": run `nvitop` in one tmux pane while your
[`ollama`](../ollama/) / [`llama.cpp`](../llama.cpp/) /
[`vllm`](../vllm/) / [`sglang`](../sglang/) /
[`text-generation-inference`](../text-generation-inference/)
serves in another, and the GPU-util / memory bars tell
you whether you are compute-bound, memory-bandwidth-
bound, or just have the wrong context length.

## Vs Already Cataloged

- **Vs [`gtop`](../gtop/) / [`bottom`](../bottom/) /
  [`btop`](../btop/) / [`glances`](../glances/) /
  [`mactop`](../mactop/):** orthogonal — those are
  whole-system monitors (CPU + RAM + disk + net + a
  bit of GPU). `nvitop` is GPU-specialist with
  per-process attribution and a kill key. Use a
  general-purpose tool for the system, `nvitop` when
  the GPU is the load-bearing resource.
- **Vs `nvidia-smi`:** the same niche, but `nvitop`
  is interactive; `nvidia-smi` is one-shot. `nvidia-
  smi -l 1` approximates the refresh, but lacks
  process-row navigation, sorting, filtering, and
  the kill key. Both call the same NVML; pick by
  whether you want to *look* (nvitop) or to *script*
  (nvidia-smi --query-gpu=...).
- **Vs [`vllm`](../vllm/) / [`text-generation-inference`](../text-generation-inference/) /
  [`sglang`](../sglang/) / [`llama.cpp`](../llama.cpp/) /
  [`ollama`](../ollama/):** orthogonal — those *use*
  the GPU; `nvitop` *watches* what they did to it.
  The intended pairing is one of those serving in one
  tmux pane, `nvitop` in another.
- **Vs [`mlflow`](../mlflow/) / [`langfuse`](../langfuse/) /
  [`weave`](../weave/):** orthogonal — those track
  experiments / traces / prompts. `nvitop` (TUI) +
  `nvitop` (Python API logging into MLflow per step)
  is the *source* of GPU-side metrics those
  dashboards then visualize.

## Caveats

- **NVIDIA only.** AMD ROCm / Apple Metal / Intel
  Arc are not supported — there is no NVML on those
  platforms. AMD users want
  [`amdgpu_top`](https://github.com/Umio-Yasuno/amdgpu_top);
  Apple Silicon users want [`mactop`](../mactop/) or
  `asitop`; Intel Arc users want `intel_gpu_top`.
- **Per-process accounting requires non-MIG mode for
  full attribution.** On A100 / H100 with MIG
  partitioning enabled, NVML still reports per-instance
  utilization but per-process attribution to a
  specific MIG slice has data-source caveats. The
  device-level totals are always correct.
- **The `K` (kill) key is exactly as dangerous as it
  sounds.** It SIGKILLs the focused PID. There is a
  confirmation prompt, but in a busy training-cluster
  context, double-check the PID before pressing
  Enter — killing the wrong job is real downtime.
- **Driver version matters.** `nvitop` follows
  `nvidia-ml-py`, which tracks the driver. Brand-new
  driver releases (e.g. 580.x in mid-2025) sometimes
  expose new fields the in-tree NVML wrapper hasn't
  bound yet — `nvitop` upstream usually lands the
  bump within days; pin a specific `nvitop` version
  in CI to avoid mid-release surprises.
- **TUI assumes a Unicode-capable terminal with at
  least 80 columns.** Below that, use `-m compact`
  or `--once`.
- **Apache-2.0 license** — permissive; safe to embed
  the Python API inside a proprietary training stack
  with attribution.
