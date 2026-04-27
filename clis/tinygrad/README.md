# tinygrad

> Snapshot date: 2026-04. Upstream: <https://github.com/tinygrad/tinygrad>

A small, hackable deep-learning framework — "between PyTorch and
[karpathy/micrograd](https://github.com/karpathy/micrograd)" — whose
explicit goal is to be the **simplest framework that still trains
real models on real accelerators**. The whole tensor library and
autograd engine fit in a few thousand lines of Python that a single
engineer can hold in their head, and the same source compiles graphs
down to CUDA, HIP (AMD), Metal (Apple), WebGPU, OpenCL, and a CPU
fallback through a uniform IR called UOps.

It is the catalog's reference for **"a deep-learning framework you
can read end-to-end in a weekend"** — useful both as an alternative
backend for inference (tinygrad + AMD ROCm beats stock PyTorch on
some MI300X kernels) and as a teaching vehicle for understanding
how PyTorch-shaped frameworks lower to GPU code.

## 1. Install footprint

- `pip install tinygrad` (Python 3.10+).
- ~5 MB pure-Python wheel; the framework itself has no compiled
  extensions. Backend-specific deps (PyOpenCL for OpenCL,
  `pyobjc-framework-Metal` for Metal, the CUDA toolkit for CUDA,
  ROCm for HIP) are picked up at runtime if present.
- Backends: CUDA (NVIDIA), HIP (AMD ROCm), Metal (Apple Silicon),
  WebGPU, OpenCL, LLVM, CLANG (CPU). Set `CUDA=1`, `HIP=1`,
  `METAL=1`, `CLANG=1`, etc. as env vars to force the backend.
- Optional `pip install tinygrad[testing]` for the model zoo
  (LLaMA / Stable Diffusion / Whisper / ResNet / EfficientNet
  reference implementations under `examples/`).

## 2. Repo, version, license

- Repo: <https://github.com/tinygrad/tinygrad>
- Version checked: **v0.12.0** (released 2026-01-12).
- HEAD pinned at this snapshot:
  `eeb8d5eb0c5ea57a839b71853064800a101cad94`.
- License: MIT. License file at
  [`LICENSE`](https://github.com/tinygrad/tinygrad/blob/master/LICENSE)
  (blob sha `d42cf56fcd134d5b3b72bd95e645bd0fd8779d09`).

## 3. What it actually does

tinygrad is shaped like PyTorch on the surface — `Tensor.randn(N,
D).matmul(W) + b`, `loss.backward()`, `optim.SGD(...)` — but the
implementation is a pipeline:

1. **Frontend.** `Tensor` operations build a lazy graph of high-level
   ops (movement, reduction, elementwise, contraction).
2. **Lowering.** The graph rewrites into UOps — a small, RISC-like
   IR with constructs for ALU, load, store, range, special. About
   30 opcodes total.
3. **Scheduler.** Fuses kernels (`schedule.py`), allocates buffers,
   resolves shape symbolics.
4. **Backend.** Each backend (CUDA / HIP / Metal / WebGPU / CLANG /
   LLVM / OpenCL) is a code generator that emits its native source
   from UOps. Compiled kernels are cached on disk
   (`~/.cache/tinygrad`).
5. **BEAM search (optional).** `BEAM=2 python train.py` runs a
   beam-search-style autotuner over kernel shapes / unrolls /
   threadgroup sizes; the winning kernel is cached.

The **entire pipeline** is < 10k lines. Reading `tensor.py` →
`ops.py` → `engine/schedule.py` → `engine/realize.py` →
`runtime/ops_metal.py` (or whichever backend) is a tractable
afternoon and the canonical way to learn how a deep-learning
framework actually works under the hood.

## 4. MCP support

None — it's a framework, not an agent. The relevant integration
boundary is "load a `safetensors` checkpoint and run inference",
which `examples/llama3.py` and `examples/sdxl.py` demonstrate.

## 5. Sub-agent model

N/A. Concurrency unit is the **kernel** (one fused UOp graph → one
GPU dispatch); training data parallelism is via the multi-device
`Device` abstraction (`Tensor.shard(devices, axis=0)`).

## 6. Telemetry stance

Off. No analytics, no opt-in callbacks. Egress is the model /
dataset URLs you `wget` in `examples/`, the kernel cache writes to
`~/.cache/tinygrad`, and whatever GPU vendor driver does on its own
(NVIDIA / AMD / Apple).

## 7. Token / context strategy

N/A at the framework level — but the bundled `examples/llama3.py`
and `examples/llama.py` demonstrate paged-attention-shaped KV cache
management hand-rolled in tinygrad ops; the result is a single-file
LLaMA-3 inference script (~600 lines) with no PyTorch / vLLM
dependency, useful as a reference for what KV-cache-aware decoding
looks like at the tensor-op level.

## 8. Hot keybinds

None — library only. The interesting "hotkeys" are env vars:

- `DEBUG=2` — print the UOp graph for every realized kernel
- `DEBUG=4` — print the generated backend source (CUDA C / Metal
  shading language / WebGPU WGSL / etc.)
- `BEAM=2` — turn on the beam-search autotuner
- `JIT=0` — disable kernel caching (debugging)
- `NOOPT=1` — disable the kernel optimiser

## 9. Killer feature, weakness, when to choose

**Killer feature.** A working PyTorch-shaped DL framework — with
autograd, a JIT, a beam-search autotuner, multi-device support, and
backends for CUDA / HIP / Metal / WebGPU / OpenCL / CPU — that you
can read end-to-end in a weekend. The whole UOp IR is ~30 opcodes;
each backend is one `runtime/ops_<name>.py` file you can swap or
add. This is the catalog's only entry where "fork the framework"
is a sensible answer to "the framework doesn't do X".

**Weakness.** v0.12.0 is still pre-1.0 and the API churns between
minor versions — `Tensor.realize()`, `TinyJit`, `nn.optim` shapes
have all moved in the last year. Pin a commit, not the tag, for
production. Performance is competitive on AMD ROCm and Apple
Metal but trails PyTorch + cuBLAS / cuDNN on NVIDIA for typical
training workloads (BEAM autotune narrows the gap on inference but
costs minutes of warm-up). The model zoo under `examples/` is
reference-quality, not a polished `transformers`-style API — porting
a new HF checkpoint is still real work.

**When to choose.**
- You want to **understand** how a deep-learning framework lowers
  Python tensor ops to GPU kernels — tinygrad is the only
  production framework small enough to read end-to-end.
- You're targeting **AMD ROCm or Apple Metal** for inference and
  the PyTorch backend story is rough — tinygrad's HIP and Metal
  backends are first-class and often faster on those substrates.
- You're shipping to **WebGPU** (browser-side ML) and want a
  framework that already knows how to emit WGSL kernels rather
  than an ONNX-to-WebGPU runtime conversion step.
- You want a **single dependency-free Python file** that runs LLaMA
  or Stable Diffusion inference for an air-gapped box.

**When to skip.**
- You're training real models at scale on NVIDIA — PyTorch +
  `torch.compile` + FlashAttention 3 + DeepSpeed is the boring
  correct answer.
- You need a wide ecosystem of pretrained models with `from_pretrained`
  ergonomics — use `transformers` + `diffusers`.
- You need long-term API stability across a 2-year project — pre-1.0
  framework churn will burn time.

## 10. Compared to neighbors in the catalog

| Tool | Primary job | Backend reach | Lines of source you can read | Ships pretrained zoo |
|------|-------------|---------------|------------------------------|----------------------|
| tinygrad | Hackable DL framework | CUDA / HIP / Metal / WebGPU / OpenCL / CPU | **~10k (whole framework)** | Reference under `examples/` |
| [candle](../candle/) | Rust ML framework | CUDA / Metal / CPU | ~30k | Yes (`candle-transformers` crate) |
| [mlx-lm](../mlx-lm/) | Apple Silicon LLM serving | Metal only | ~5k (the LM wrapper) | Yes (HF `mlx-community`) |
| [llama.cpp](../llama.cpp/) | C++ LLM inference | CUDA / Metal / CPU + many | ~200k | GGUF ecosystem |

Decision shortcut:
- "Read a DL framework end-to-end" → **tinygrad**.
- "Same idea but in Rust + production-shaped" → [`candle`](../candle/).
- "Best Apple Silicon LLM perf, don't care to read source" → [`mlx-lm`](../mlx-lm/).
- "Inference only, every quant under the sun" → [`llama.cpp`](../llama.cpp/).
