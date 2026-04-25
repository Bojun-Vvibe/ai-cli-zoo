# axolotl

> Snapshot date: 2026-04. Upstream: <https://github.com/axolotl-ai-cloud/axolotl>

"**Go ahead and axolotl questions.**" Axolotl is the **YAML-driven
fine-tuning workhorse** for open-weight LLMs: one config file
declares base model, dataset(s) + chat template, tuning method
(full / LoRA / QLoRA / ReLoRA / GPTQ-LoRA / DPO / KTO / ORPO /
SimPO / RLHF / multi-modal), precision (bf16 / fp16 / fp8), and
distribution strategy (DeepSpeed ZeRO-1/2/3, FSDP, FSDP+QLoRA,
multi-node), then `axolotl train config.yml` runs the whole
pipeline end-to-end on top of the HF `transformers` + `trl` +
`peft` + `accelerate` stack — including dataset prep, tokenization
caching, gradient checkpointing, sample packing, Liger / Flash
Attention 2 / xformers kernels, LoRA merging, and HF Hub push.

## 1. Install footprint

- `pip install axolotl` — installs the `axolotl` CLI plus a
  pinned set of HF / torch deps (CUDA wheels assumed). Heavy
  install (~3-5 GB resolved) because it pulls torch, bitsandbytes,
  flash-attn build deps, deepspeed, trl, peft, transformers.
- Docker: `axolotlai/axolotl:main-latest` for a known-good CUDA
  + flash-attn + deepspeed image.
- One CLI: `axolotl preprocess`, `axolotl train`, `axolotl
  inference`, `axolotl evaluate`, `axolotl merge-lora`,
  `axolotl merge-sharded-fsdp-weights`, `axolotl lm-eval`,
  `axolotl fetch examples`.
- Python ≥ 3.11, CUDA ≥ 12.1; CPU-only inference works,
  CPU-only training does not. Apple Silicon training is
  unsupported (NVIDIA-first; AMD ROCm is experimental).

## 2. Repo + version + license

- Repo: <https://github.com/axolotl-ai-cloud/axolotl>
- Latest release: **v0.16.1** (2026-04-02)
- License: **Apache-2.0** —
  <https://github.com/axolotl-ai-cloud/axolotl/blob/main/LICENSE>
- HEAD SHA: `798c8fba897ceb2e238dfaac9bb1731d2f1cacab`
- Default branch: `main`
- Language: Python (~12k stars)

## 3. Models supported

Llama 2 / 3 / 3.1 / 3.2 / 3.3 / 4, Mistral / Mixtral / Mistral-Nemo,
Qwen 1.5 / 2 / 2.5 / 3, Qwen-VL, Gemma 1 / 2 / 3, Phi-2 / 3 / 4,
DeepSeek-V2 / V3 / Coder / R1, Yi, Falcon, MPT, StableLM, GPT-NeoX,
ChatGLM / GLM-4, Cohere Command-R / R+, Granite, OLMo, Pixtral,
LLaVA / LLaVA-Next, Idefics2 / 3, plus any HF causal-LM that fits
the `transformers` `AutoModelForCausalLM` shape. Multi-modal
fine-tunes (vision + text) are first-class with dedicated
`mm_plugin` configs.

## 4. Simple usage

```bash
# 1. grab a known-good example config
axolotl fetch examples
# -> ./examples/llama-3/lora-1b.yml

# 2. (optional) preprocess + cache the tokenized dataset
axolotl preprocess examples/llama-3/lora-1b.yml

# 3. train (single-GPU; for multi-GPU just prepend
#    `accelerate launch --multi_gpu --num_processes 4`)
axolotl train examples/llama-3/lora-1b.yml

# 4. merge the LoRA back into the base for serving
axolotl merge-lora examples/llama-3/lora-1b.yml \
  --lora-model-dir ./outputs/llama-3-1b-lora

# 5. quick sanity-check inference against the merged model
axolotl inference examples/llama-3/lora-1b.yml \
  --base-model ./outputs/llama-3-1b-lora-merged
```

## 5. Why it's interesting

- **YAML *is* the experiment**: base model, dataset(s), chat
  template, tuning method, optimizer, scheduler, precision,
  distribution strategy, eval harness — all in one diffable file
  that round-trips into a HF Hub model card. Re-running someone
  else's fine-tune is "clone repo, `axolotl train their.yml`".
- **Tuning method menu under one CLI**: full SFT, LoRA, QLoRA,
  ReLoRA, GPTQ-LoRA, DPO, KTO, ORPO, SimPO, IPO, RLOO, plus
  multi-modal SFT — same `axolotl train` entry point, just a
  different `rl:` / `adapter:` block in the YAML.
- **Distribution is a config flag, not a rewrite**: DeepSpeed
  ZeRO-1/2/3, FSDP, FSDP+QLoRA, sequence parallelism, tensor
  parallel, multi-node — flip `deepspeed:` or `fsdp_config:` and
  the same config scales from a single 24 GB consumer card to a
  multi-node H100 cluster.
- **Throughput knobs are first-class**: sample packing, Liger
  kernels, Flash Attention 2, xformers, paged optimizers,
  gradient checkpointing, fused cross-entropy — all toggles
  in YAML, no kernel patching required.

## 6. Caveats

- **CUDA-only in practice** — flash-attn, bitsandbytes, deepspeed
  presume NVIDIA; ROCm is experimental and Apple Silicon training
  is not a supported path.
- **Heavy install** with a sharp version matrix between torch /
  CUDA / flash-attn / bitsandbytes / deepspeed; use the official
  Docker image to avoid a half-day of dep-pinning.
- The YAML surface is huge (hundreds of keys); reading
  `examples/` is mandatory before authoring a config from
  scratch.
- Axolotl trains and merges weights but does *not* serve them —
  pair with a serving CLI (vLLM, lmdeploy, sglang, litserve) for
  the deploy step.
