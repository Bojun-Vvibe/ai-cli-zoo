# peft

> Snapshot date: 2026-04. Upstream: <https://github.com/huggingface/peft>

**Hugging Face's parameter-efficient fine-tuning library — the canonical implementation of LoRA / QLoRA / DoRA / VeRA / IA3 / OFT / BOFT / LoHa / LoKr / X-LoRA / HRA / FourierFT / VBLoRA / RandLoRA / Prompt-Tuning / P-Tuning / Prefix-Tuning / Adaption-Prompt / MultiTask-Prompt-Tuning.**
`peft` is the layer the rest of the post-training stack
(`transformers`, [`trl`](../trl/), [`unsloth`](../unsloth/),
[`axolotl`](../axolotl/), [`diffusers`], `accelerate`) calls when it
needs to inject low-rank adapters into an existing pretrained model
without touching the base weights. The contract is small —
`get_peft_model(model, LoraConfig(...))` returns a wrapped model
where only the adapter parameters are trainable, the base is
frozen, and `model.save_pretrained("./out")` writes only the
adapter delta — but that contract is the substrate for nearly
every "fine-tune Llama-3 on one GPU" demo on the internet.
v0.19.x landed in **2026-04** with continued additions for the
non-LoRA adapter family (BoNE, MiSS, OSF, RoAd) and tighter
DeepSpeed-ZeRO-3 + FSDP2 + TP integration.

## 1. Footprint

- Pure Python library; no CLI of its own — used through the
  `transformers` `Trainer`, [`trl`](../trl/) trainers, or directly
  from a custom training loop.
- Adapters are **stackable and switchable at runtime** —
  `model.add_adapter("law", lora_law_cfg)`,
  `model.add_adapter("medical", lora_med_cfg)`,
  `model.set_adapter("law")` swaps which adapter the next forward
  pass uses. Multi-adapter weighted merging
  (`model.add_weighted_adapter([...])`) builds task-mixture
  adapters at inference.
- Python ≥ 3.9, PyTorch ≥ 2.4. CUDA / ROCm / Apple-Silicon-MPS /
  CPU all work for the pure-PyTorch adapters; `bitsandbytes` 4-bit
  / 8-bit base quantisation (the QLoRA path) is NVIDIA-first.

## 2. Repo + version + license

- Repo: <https://github.com/huggingface/peft>
- Latest release: **`v0.19.1`** (2026-04-16)
- HEAD on `main`: `852019c` (2026-04-25)
- License: **Apache-2.0** —
  <https://github.com/huggingface/peft/blob/main/LICENSE>
- License path in repo: `LICENSE`
- License blob SHA: `261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64`
- License SPDX: `Apache-2.0`
- Default branch: `main`
- Language: Python

## 3. Methods shipped

- **Low-rank reparam family**: LoRA, QLoRA (LoRA + 4-bit base),
  DoRA (weight-decomposed LoRA), VeRA (vector random LoRA), LoHa
  (Hadamard product), LoKr (Kronecker product), AdaLoRA (rank
  adaptive), X-LoRA (mixture-of-experts over LoRA adapters),
  RandLoRA, VBLoRA, FourierFT, HRA, BoNE, MiSS, OSF, RoAd.
- **Bias / scale family**: IA3 (rescale activations), BitFit (bias
  only), LN-Tuning (layernorm only).
- **Orthogonal / rotation family**: OFT (orthogonal fine-tuning),
  BOFT (butterfly OFT), C-OFT.
- **Soft-prompt family**: Prompt-Tuning, P-Tuning, P-Tuning-v2,
  Prefix-Tuning, MultiTask-Prompt-Tuning, Adaption-Prompt
  (Llama-Adapter / Llama-Adapter-v2).
- **Mixture / routing**: PolyTuning, X-LoRA expert-mixing.
- **Modality**: works on `transformers` causal-LM / seq2seq /
  encoder, `diffusers` UNet / DiT / Transformer2DModel (LoRA on
  Stable Diffusion / SDXL / Flux / SD3 attention layers), Whisper
  / Wav2Vec2 audio encoders, vision encoders (ViT / CLIP / DINO).

## 4. Simple usage

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import LoraConfig, get_peft_model, TaskType

base = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-3B")
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-3B")

cfg = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    use_dora=True,                    # DoRA on top of LoRA
)
model = get_peft_model(base, cfg)
model.print_trainable_parameters()
# trainable params: ~16M / ~3B (~0.5%)

# now hand `model` to transformers.Trainer or trl.SFTTrainer

# ----- inference: load the adapter on top of the base -----
from peft import PeftModel
base = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-3B")
model = PeftModel.from_pretrained(base, "./out")
# or merge into the base for deployment
model = model.merge_and_unload()
```

## 5. Why it's interesting

- **The reference implementation everyone else wraps** — Unsloth's
  fused LoRA kernel, Axolotl's YAML, [`trl`](../trl/)'s
  `peft_config=` argument, `diffusers`'s LoRA loader,
  [`vllm`](../vllm/)'s `--enable-lora`, [`lorax`](../lorax/)'s
  per-request adapter dispatch, [`text-generation-inference`](../text-generation-inference/)'s
  multi-LoRA serving — all consume `peft`'s `LoraConfig` /
  `safetensors` adapter format. The format is the de-facto standard
  for "a LoRA adapter on Hugging Face Hub".
- **Adapter swap at runtime is the killer feature** —
  `model.set_adapter("customer_support")` flips behaviour without
  reloading the base; multi-tenant SaaS serves N customers from
  one base model with N tiny adapter files. `lorax` and `vllm`'s
  multi-LoRA paths productionise exactly this primitive.
- **DoRA / VeRA / X-LoRA / FourierFT close the LoRA-vs-full-tuning
  quality gap** — DoRA's weight-decomposed update consistently
  matches full fine-tuning on commonsense + math benchmarks at the
  same parameter budget; X-LoRA mixtures route between domain
  adapters per-token; VeRA cuts adapter parameters another 10x
  with a frozen-random-LoRA + scaling-vector trick.
- **QLoRA is one config flag** — load the base with
  `bitsandbytes` 4-bit (`BitsAndBytesConfig(load_in_4bit=True,
  bnb_4bit_quant_type="nf4")`) then `get_peft_model(...)` and
  the rest of the training script is unchanged; the recipe behind
  every "fine-tune a 70B model on a single 80GB H100" notebook.
- **Diffusers integration is in-tree** — same `LoraConfig` /
  `get_peft_model` API works on Stable Diffusion / SDXL / Flux
  / SD3 transformer blocks; the LoRA files HuggingFace's image
  ecosystem trades are written by this library.

## 6. Caveats

- The non-LoRA adapter zoo (OFT, BOFT, FourierFT, HRA, VeRA,
  RandLoRA, BoNE, MiSS) is **research-grade** — the public
  literature on which beats LoRA on which task is still in flux,
  and downstream tooling (vLLM multi-LoRA, lorax, TGI) only
  understands LoRA / DoRA at the serving layer. Default to LoRA
  / DoRA / QLoRA unless you have a specific paper in mind.
- `merge_and_unload()` produces a base-shape checkpoint that any
  inference engine accepts, but **once merged the adapter
  identity is lost** — keep the unmerged adapter for runtime
  swap / Lorax-style multi-tenant serving.
- The `target_modules` list is **architecture-specific** — wrong
  module names silently train zero parameters (the trainable-
  param count goes to 0). `peft` ships a small registry of common
  defaults but custom architectures need an explicit list. The
  `TaskType` matters too (`CAUSAL_LM` vs `SEQ_2_SEQ_LM` vs
  `TOKEN_CLS` vs `FEATURE_EXTRACTION`).
- DeepSpeed-ZeRO-3 + adapter saving has historical sharp edges —
  v0.19+ resolves most of them but pin a known-good combo
  (`peft` + `accelerate` + `transformers` + `deepspeed`) for
  multi-node runs.

## 7. Where it sits in the catalog

- **Pair with**: [`trl`](../trl/) (`peft_config=` slot in every
  trainer), [`unsloth`](../unsloth/) (faster fused-LoRA kernels,
  same `LoraConfig`), [`axolotl`](../axolotl/) (YAML resolves to
  `peft` configs), [`vllm`](../vllm/) / [`lorax`](../lorax/) /
  [`text-generation-inference`](../text-generation-inference/)
  (multi-adapter serving consuming `peft`-format adapters),
  [`mergekit`](../mergekit/) (merge `peft`-shaped adapters into
  base or into other adapters), [`huggingface-cli`](../huggingface-cli/)
  (pull / push adapters from Hub).
- **Don't use for**: full fine-tuning (just use
  `transformers.Trainer` directly); pretraining (Nanotron /
  Megatron-LM / [`litgpt`](../litgpt/)); RL post-training without
  SFT first ([`trl`](../trl/)'s `GRPOTrainer` accepts a `peft`
  adapter as the policy).
