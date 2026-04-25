# unsloth

> Snapshot date: 2026-04. Upstream: <https://github.com/unslothai/unsloth>

"**Finetune & RL LLMs 2× faster with 70% less memory.**" Unsloth is
a Python library + thin CLI surface for parameter-efficient
fine-tuning (LoRA / QLoRA / full / continued pre-training / DPO /
ORPO / GRPO / KTO) of open-weight LLMs and VLMs. It rewrites the
hot paths of the HuggingFace `transformers` + `peft` + `trl` stack
in hand-tuned Triton kernels (RoPE, RMSNorm, cross-entropy, MLP,
attention, fast LoRA matmul) so a 7-13B model fits and trains on a
single 16-24 GB consumer GPU. The recent v0.1.37-beta line ships
the redesigned UI plus first-class Qwen3.6 / Gemma-3 / Llama-3.x /
DeepSeek-R1-distill / Mistral / Phi-4 / gpt-oss recipes.

## 1. Install footprint

- `pip install unsloth` (latest pip wheel; pulls a matched
  `xformers` + `triton` + `bitsandbytes` + `transformers` + `trl` +
  `peft` + `accelerate` set).
- Conda: `conda install -c conda-forge unsloth`.
- No standalone `unsloth` binary — the surface is `python -m
  unsloth.cli` plus `from unsloth import FastLanguageModel` in
  notebook / script form. Hundreds of ready-to-run Colab / Kaggle
  notebooks under `unsloth-notebooks/`.
- Python ≥ 3.9, CUDA ≥ 11.8 (Linux / WSL2). NVIDIA only for the
  fast kernels; CPU import works for inspection.

## 2. Repo + version + license

- Repo: <https://github.com/unslothai/unsloth>
- Latest release: **v0.1.37-beta** (2026-04-23) — "New UI Redesign +
  Qwen3.6"
- License: **Apache-2.0** —
  <https://github.com/unslothai/unsloth/blob/main/LICENSE>
- HEAD SHA: `b09aa82a3ac9ac9794c497e4b8f747f77e52b162`
- Default branch: `main`
- Language: Python

## 3. Models supported

Llama-3 / 3.1 / 3.2 / 3.3, Mistral / Mixtral, Qwen 2.5 / 3 / 3.6,
Gemma 2 / 3, Phi-3 / 4, DeepSeek-V3 / R1 / R1-distill, gpt-oss,
TinyLlama, Yi, CodeLlama, plus VLMs (Llama-3.2-Vision, Qwen-VL,
Pixtral). Anything matching the `transformers` `AutoModel*` API +
one of the patched architectures.

## 4. Simple usage

```python
from unsloth import FastLanguageModel
import torch

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name = "unsloth/Llama-3.2-3B-Instruct",
    max_seq_length = 4096,
    load_in_4bit = True,
)

model = FastLanguageModel.get_peft_model(
    model, r = 16, lora_alpha = 16,
    target_modules = ["q_proj","k_proj","v_proj","o_proj",
                      "gate_proj","up_proj","down_proj"],
)

# now hand `model` to a standard `trl.SFTTrainer` /
# `DPOTrainer` / `GRPOTrainer` — Unsloth has monkey-patched
# the kernels underneath.
```

## 5. Why it's interesting

- **Kernel-level rewrite, not a wrapper**: hand-written Triton for
  RoPE, RMSNorm, cross-entropy, MLP, attention, plus a fused LoRA
  matmul that drops VRAM ~70% and lifts throughput ~2× vs vanilla
  HF + bitsandbytes — measured against the same training script,
  not a synthetic benchmark.
- **Drop-in for `trl`**: `FastLanguageModel.get_peft_model(...)`
  returns a model the upstream `SFTTrainer` / `DPOTrainer` /
  `GRPOTrainer` / `ORPOTrainer` / `KTOTrainer` already accept, so
  you keep the standard HuggingFace post-training surface and just
  swap in a faster engine.
- **Recipe library as the docs**: 100+ versioned Colab notebooks
  (one per model × per technique) in `unsloth-notebooks/` are the
  primary documentation — copy a notebook, change the dataset,
  ship a fine-tune. New model day-zero support lands as a notebook
  before it lands as a doc page.
- **Single-GPU as the design target**: every default (4-bit base,
  rank-16 LoRA, gradient checkpointing on, packing on, fused
  cross-entropy) is tuned for "this has to fit on one 24 GB
  card" — the antidote to multi-node deepspeed configs for solo
  practitioners.

## 6. Caveats

- Fast path is **NVIDIA + CUDA only**; AMD / Apple Silicon work
  through the unpatched HF fallback (no speedup).
- Versioned tightly to specific `transformers` / `trl` / `peft`
  releases — `pip install -U unsloth` may pin you to slightly
  older HF; check `pyproject.toml` of the release tag.
- Beta cadence is fast (multiple `vX.Y.Z-beta` per month); pin a
  SHA in production training scripts.
