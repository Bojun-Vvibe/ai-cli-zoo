# liger-kernel

> Snapshot date: 2026-04-26. Upstream: <https://github.com/linkedin/Liger-Kernel>

"**Efficient Triton Kernels for LLM Training.**" Liger-Kernel
is LinkedIn's drop-in collection of fused Triton kernels for
the hottest layers in transformer training — RMSNorm, RoPE,
SwiGLU, GeGLU, CrossEntropy, FusedLinearCrossEntropy, KL/JSD
distillation losses, DPO/ORPO/SimPO/CPO preference losses —
that monkey-patches a Hugging Face model in one line and
delivers ≈20% higher throughput and ≈60% lower peak memory
on multi-GPU training without changing the training loop.

## 1. Install footprint

- Library: `pip install liger-kernel` (CUDA / Triton wheel) or
  `pip install liger-kernel-nightly`; AMD ROCm and Intel XPU
  wheels available; Apple MPS via the pure-PyTorch fallback.
- No CLI binary — used as `from liger_kernel.transformers
  import apply_liger_kernel_to_llama; apply_liger_kernel_to_llama()`
  before instantiating an HF `AutoModelForCausalLM`, or via
  the `AutoLigerKernelForCausalLM` wrapper that picks the
  right patcher from the model's `config.model_type`.
- Plays inside any HF Trainer / Accelerate / DeepSpeed / FSDP /
  PyTorch-Lightning / Axolotl / [`unsloth`](../unsloth/) /
  [`llama-factory`](../llama-factory/) /
  [`trl`](https://github.com/huggingface/trl) loop with no other
  changes.

## 2. Repo + version + license

- Repo: <https://github.com/linkedin/Liger-Kernel>
- Latest release: **v0.7.0** (published 2026-02-12)
- License: **BSD-2-Clause** —
  <https://github.com/linkedin/Liger-Kernel/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (Triton)
- Stars: ~6.3k

## 3. Models supported

- One-line patchers for: Llama 2 / 3 / 3.1 / 3.2 / 4, Mistral,
  Mixtral, Gemma 1 / 2 / 3, Qwen 2 / 2.5 / 3 (incl. Qwen2-VL,
  Qwen2.5-VL, Qwen3-MoE), Phi-3 / Phi-4, Granite 3, Olmo 2,
  Glm4, Paligemma, SmolLM, plus generic `liger_kernel.transformers`
  module-level swaps for any model that uses RMSNorm + RoPE +
  SwiGLU.
- Preference-tuning losses: DPO, ORPO, SimPO, CPO, KTO — all
  implemented as fused linear+loss kernels that skip
  materializing the full logits tensor (the dominant memory
  cost of long-context preference training).
- Distillation losses: KLDiv, JSD, fused linear cross-entropy
  for teacher-student training.
- Compatible with bf16, fp16, fp8 (Triton autotune), tensor
  parallel, sequence parallel, and gradient checkpointing.

## 4. Notable angle

**It is the rare "performance kernel" library that is also a
single import away.** Where [`unsloth`](../unsloth/) bundles
its own training stack, custom model classes, and a "free
notebook" UX, and where DeepSpeed / Megatron-LM ask you to
adopt their entire training framework, Liger-Kernel does the
opposite: it leaves your HF model class, your HF Trainer, and
your existing Accelerate / FSDP / DeepSpeed / PyTorch-Lightning
config alone, and patches the four or five hot ops in place
with a single `apply_liger_kernel_to_<model>()` call. The
fused-linear-cross-entropy and fused-linear-DPO kernels in
particular avoid ever materializing the `[B, S, V]` logits
tensor, which is the practical reason teams report fitting
Llama-3-70B SFT on 8×H100 instead of 16, and the reason the
preference-tuning recipes in [`trl`](https://github.com/huggingface/trl)
and [`llama-factory`](../llama-factory/) increasingly default
to Liger when CUDA is available. The trade-off: it is
PyTorch + Triton + Hugging Face only — no JAX, no MLX, no
inference-time use ([`vllm`](../vllm/) / [`sglang`](../sglang/)
have their own kernels) — and the kernels are tied to specific
op signatures, so a brand-new architecture may need to wait a
release for its patcher. Use it when you are training or
fine-tuning a supported HF model on NVIDIA / AMD / XPU and
want a zero-config 20% / 60% throughput / memory win; skip it
when you are doing inference, JAX/MLX training, or running on
a model whose architecture Liger has not patched yet.

## 5. Last verified

2026-04-26 via `gh api repos/linkedin/Liger-Kernel` and
`gh api repos/linkedin/Liger-Kernel/releases/latest` → v0.7.0
(2026-02-12), license BSD-2-Clause at
<https://github.com/linkedin/Liger-Kernel/blob/main/LICENSE>.
