# litgpt

> Snapshot date: 2026-04. Upstream: <https://github.com/Lightning-AI/litgpt>
> License file: <https://github.com/Lightning-AI/litgpt/blob/main/LICENSE>
> Pinned: `v0.5.12` (2025-12-18). `main` continues to receive commits and is
> the source for the published Studios; pin the tag in CI.

A single-package Python CLI from Lightning AI that exposes the **full
LLM lifecycle — pretrain, finetune (full / LoRA / QLoRA / Adapter), eval,
quantize, serve, chat, and convert-to-HF — as one cohesive `litgpt
<verb>` surface**. Built on PyTorch + Lightning Fabric, with a curated
catalog of 20+ open-weight model architectures whose forward passes are
implemented in readable, hackable PyTorch (no `transformers` modeling
files, no monkey-patches) so the same code is the one you train *and*
the one you read to understand the model.

It occupies a different niche from [`axolotl`](../axolotl/) and
[`unsloth`](../unsloth/): axolotl is a YAML-driven training framework
on top of HuggingFace; unsloth is a memory/speed optimisation library
that wraps HuggingFace; **litgpt is a from-scratch reimplementation**
of each supported architecture, so the entire training and inference
path is one Python codebase you can read in an afternoon.

## 1. Install footprint

- `pip install 'litgpt[all]'` — full surface (training + serving +
  quantize + eval). Pulls in `torch`, `lightning`, `jsonargparse`,
  `huggingface-hub`, `safetensors`, `bitsandbytes`,
  `lm-eval-harness`, `litserve`.
- `pip install litgpt` — minimal: lets you `litgpt download` and
  `litgpt chat` but not the LoRA / eval / serve verbs.
- Disk: the binary is small; the storage cost is the model weights.
  Pretraining recipes assume ≥80 GB GPU(s); LoRA finetuning of 7B
  models runs on a single 24 GB consumer GPU; QLoRA brings 13B
  finetuning into the same 24 GB envelope.
- Config is **CLI flags + an optional `config.yaml`** parsed by
  jsonargparse, which means every flag has an equivalent YAML key
  and you can mix the two (`litgpt finetune lora --config base.yaml
  --train.epochs 3`).

## 2. License

Apache-2.0.

Lightning AI also publishes commercial Lightning Studios that bundle
litgpt with managed infrastructure; the OSS CLI is unencumbered
Apache-2.0, no separate commercial license is needed for production
use.

## 3. Models supported

The hand-maintained model catalog (each is a separate
`litgpt/model_<name>.py` with a from-scratch PyTorch implementation):

- **Llama family** — Llama 2 (7/13/70B), Llama 3 / 3.1 / 3.2 / 3.3
  (incl. 8B / 70B / 405B), CodeLlama
- **Mistral / Mixtral** — Mistral 7B, Mistral Nemo, Mixtral 8x7B,
  Mixtral 8x22B (sparse MoE forward pass implemented natively)
- **Gemma 1 / 2 / 3** (incl. 2B / 7B / 9B / 27B sizes)
- **Phi 1.5 / 2 / 3 / 3.5 / 4**
- **Qwen 2 / 2.5** (Coder, Math, dense and MoE variants)
- **DeepSeek V2 / V3 / R1 distilled dense variants**
- **Falcon, StableLM, TinyLlama, Pythia, OLMo, OpenLLaMA, Vicuna,
  RedPajama, OpenChat, Dolly, FreeWilly, NousHermes, Platypus**

`litgpt download list` prints the full registry against the live
HuggingFace mirror; `litgpt download <hf-name>` pulls weights and
runs the conversion to litgpt's checkpoint format (a single
`lit_model.pth` plus `config.json` and `tokenizer.json`).

For inference / fine-tuning over models *not* in the catalog, you
either add the architecture file (a few hundred lines of PyTorch) or
use a different tool — there is no `AutoModelForCausalLM` wildcard
loader by design.

## 4. MCP support

**No.** litgpt is a model lifecycle tool, not an agent. `litgpt
serve` exposes an OpenAI-compatible HTTP endpoint (it shells out to
[`litserve`](../litserve/) under the hood); for MCP, point an
MCP-aware client at that endpoint as the model backend, the same way
you would with [`vllm`](../vllm/) or [`openllm`](../openllm/).

## 5. Sub-agent model

None. Each verb is one Lightning Fabric process (or N processes in
DDP / FSDP / DeepSpeed when you pass `--devices N` or `--strategy
fsdp`). There is no planner, no tool loop, no agent — this is a
training and serving CLI, not an agent runtime.

## 6. Telemetry stance

**Off, no opt-in.** No analytics SDK in the package. Outbound network
calls are: HuggingFace Hub on `litgpt download`, optional
`wandb` / `tensorboard` logging that you opt into per-run, and the
served endpoint on `litgpt serve`. Lightning Studios (the commercial
cloud product) is a separate signup; the CLI itself does not phone
home.

## 7. Prompt-cache strategy

For training, N/A. For inference (`litgpt chat` and `litgpt serve`),
KV cache is the standard incremental-decoding cache built into the
custom forward pass; there is no cross-request prefix cache layer in
the OSS server. If you need prefix caching at scale, deploy the
finetuned weights via [`vllm`](../vllm/) or
[`sglang`](../sglang/) instead of `litgpt serve` — those engines
implement automatic prefix caching, which the litgpt server
deliberately does not.

## 8. Hot keybinds

No TUI. CLI verbs (one per subcommand of `litgpt`):

- `litgpt download <hf-name>` — pull weights from HF, convert to
  litgpt format. `litgpt download list` lists supported models.
- `litgpt chat <checkpoint-dir>` — interactive REPL against a
  downloaded or finetuned checkpoint, runs on CPU / single GPU.
- `litgpt finetune lora <checkpoint-dir> --data <data> --train.epochs
  3` — LoRA finetune with sane defaults; `finetune full`,
  `finetune adapter`, and `finetune adapter_v2` are siblings.
- `litgpt finetune lora ... --quantize bnb.nf4` — QLoRA flavour for
  consumer GPUs.
- `litgpt pretrain <model-name> --data <data> --train.global_batch_size
  N` — from-scratch pretraining, supports DDP / FSDP / TP via
  `--strategy fsdp` and `--devices N`.
- `litgpt evaluate <checkpoint-dir> --tasks
  mmlu,gsm8k,truthfulqa_mc2` — runs `lm-evaluation-harness` against
  the checkpoint without leaving the CLI.
- `litgpt serve <checkpoint-dir> --port 8000` — OpenAI-compatible
  HTTP endpoint via `litserve`, single command.
- `litgpt convert_to_litgpt <hf-dir>` and `litgpt convert_from_litgpt
  <litgpt-dir>` — the round-trip with HuggingFace format, so a
  finetuned model can be uploaded to the Hub or served by
  `transformers` / `vllm` directly.
- `litgpt merge_lora <lora-dir>` — merge a LoRA adapter back into the
  base weights to produce a single deployable checkpoint.

Built-in dataset shims for `alpaca`, `dolly`, `lima`, `flan`,
`longform`, `tinystories`, plus a generic `JSON` / `JSONL` /
`Parquet` / `text` loader for custom data, all under `--data <name>`.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **The same readable PyTorch is the training and
inference path.** Each model is one file (`litgpt/model_llama.py`,
`litgpt/model_mixtral.py`, ...) implementing the forward pass from
scratch in 200-400 lines of PyTorch — no `transformers.AutoModel`
black box, no `flash_attn` monkey-patch, no `accelerate` indirection
to chase through. When LoRA finetuning produces a numerically wrong
gradient on Mixtral, you can read the actual `forward(self, x)` of
the MoE layer, find the bug, and fix it in the same file you ship.
Combined with the lifecycle coverage (download → finetune → eval →
quantize → merge → serve as one CLI namespace) and Lightning
Fabric's clean DDP / FSDP / DeepSpeed strategy switch (`--strategy
fsdp` is one flag, not a config-file pilgrimage), litgpt is the most
*pedagogically transparent* training stack in the catalog while
still being production-shaped enough for real fine-tuning runs on
70B-class models.

**Weakness.** **The hand-maintained architecture catalog is also the
ceiling.** A new model whose architecture file has not yet landed in
litgpt cannot be loaded at all — there is no `AutoModelForCausalLM`
fallback. New architectures lag the HuggingFace ecosystem by weeks
to months. Inference performance is honest single-batch PyTorch; for
high-QPS serving you must export to [`vllm`](../vllm/) or
[`sglang`](../sglang/) (the export path is one command, but it is a
required step). The narrower `transformers` / `peft` / `bitsandbytes`
integration surface compared to [`axolotl`](../axolotl/) means
exotic optimisers or PEFT methods sometimes need to be added by
hand.

**When to choose.**

- You want to **read the actual forward pass** of the model you are
  training — debugging a Mixtral gradient or a Llama-3 RoPE
  implementation is `code .` away, not a chase through three layers
  of indirection.
- You want **one CLI for the whole lifecycle** — download → LoRA
  finetune → evaluate → merge → serve, no leaving the `litgpt`
  namespace.
- You want **multi-GPU FSDP or DeepSpeed pretraining** with a single
  `--strategy fsdp --devices 8` flag instead of writing an
  `accelerate` config.
- You want a **stable Apache-2.0 baseline** for an internal training
  fork (the codebase is small enough to actually fork and maintain).
- You are **teaching or learning** how modern LLMs work — the
  per-architecture single-file model definitions are the closest
  thing in this catalog to the nanoGPT pedagogical style at the
  scale of real production models.

**When not to choose.** You need to load a brand-new
just-released architecture today — go via HuggingFace `transformers`
+ `axolotl` / `unsloth` until litgpt's architecture file lands. You
need maximum-throughput inference serving — fine-tune with litgpt,
serve with [`vllm`](../vllm/) / [`sglang`](../sglang/) /
[`openllm`](../openllm/). You want a hosted training UX with a
managed cluster — Lightning Studios is the commercial layer; if
you are not paying Lightning, [`modal`](../modal/) +
[`axolotl`](../axolotl/) is the closer OSS analogue.
