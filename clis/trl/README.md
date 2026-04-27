# trl

> Snapshot date: 2026-04. Upstream: <https://github.com/huggingface/trl>

**Hugging Face's RL + post-training library — the canonical home of `SFTTrainer` / `DPOTrainer` / `GRPOTrainer` / `PPOTrainer` / `RewardTrainer` for transformer-shaped LMs.**
`trl` (Transformer Reinforcement Learning) is the library every other
post-training stack in the catalog — [`unsloth`](../unsloth/),
[`axolotl`](../axolotl/), [`liger-kernel`](../liger-kernel/),
[`mergekit`](../mergekit/) — composes against rather than replaces.
It owns the trainer surface for the modern RLHF / RLAIF / preference-
optimisation lineage (SFT → reward modelling → PPO / DPO / IPO / KTO
/ ORPO / SimPO / GRPO / RLOO / Online-DPO / Nash-MD / GKD / BCO),
plus the multimodal trainers (`SFTTrainer` for VLMs, `DPOTrainer` for
image-text preference data). Built on top of `transformers` +
`accelerate` + `peft` + `datasets`, distribution via DeepSpeed-ZeRO
/ FSDP / FSDP2 / TP is config-flag rather than rewrite. As of
**v1.3.0** (2026-04-26), `trl` is the post-1.0 stable surface —
trainers are API-frozen for the v1.x line, and downstream wrappers
(Unsloth, Axolotl, Llama-Factory) pin minor versions tightly.

## 1. Footprint

- Pure Python library; the trainers subclass `transformers.Trainer`
  so existing HF training muscle memory transfers.
- CLI surface: `trl sft` / `trl dpo` / `trl grpo` / `trl chat` /
  `trl env` — thin wrappers over the trainer scripts in
  `examples/scripts/` for one-shot runs without writing a Python
  driver. Real production training scripts subclass the trainer
  directly.
- Python ≥ 3.9, PyTorch ≥ 2.4. Multi-GPU via `accelerate launch`.
  CUDA / ROCm / Apple-Silicon-MPS / CPU all work; the fast kernels
  (Flash-Attention-2, Liger, Unsloth) are NVIDIA-only.

## 2. Repo + version + license

- Repo: <https://github.com/huggingface/trl>
- Latest release: **`v1.3.0`** (2026-04-26)
- HEAD on `main`: `4798893` (2026-04-26)
- License: **Apache-2.0** —
  <https://github.com/huggingface/trl/blob/main/LICENSE>
- License path in repo: `LICENSE`
- License blob SHA: `f577b7741dfb5c6af119f250971549dd7750acb9`
- License SPDX: `Apache-2.0`
- Default branch: `main`
- Language: Python

## 3. Trainers shipped

- **Supervised fine-tuning**: `SFTTrainer` — chat-template aware,
  packing, completion-only loss masking, multimodal (VLM) variant.
- **Reward modelling**: `RewardTrainer` — Bradley-Terry pairwise
  preference loss with margin support.
- **Preference optimisation**: `DPOTrainer`, `IPOTrainer`,
  `KTOTrainer`, `ORPOTrainer`, `SimPOTrainer`, `CPOTrainer`,
  `BCOTrainer`. Each one is a one-line swap on the same dataset
  shape (`{prompt, chosen, rejected}`).
- **On-policy RL**: `PPOTrainer`, `RLOOTrainer`, `GRPOTrainer`
  (the trainer behind DeepSeek-R1-Zero-style reasoning RL),
  `Online-DPOTrainer`, `Nash-MDTrainer`, `XPOTrainer`.
- **Distillation**: `GKDTrainer` (generalised knowledge
  distillation, on-policy + off-policy mix).
- **Reasoning / agentic**: `GRPOTrainer` accepts a list of
  custom `reward_funcs` so RLVR (verifiable rewards) — unit-test
  pass / regex match / math-answer correctness — is the supported
  path, not a fork.

## 4. Simple usage

```python
from datasets import load_dataset
from trl import SFTTrainer, SFTConfig

dataset = load_dataset("trl-lib/Capybara", split="train")

trainer = SFTTrainer(
    model="Qwen/Qwen2.5-3B",
    train_dataset=dataset,
    args=SFTConfig(
        output_dir="qwen2.5-3b-capybara",
        max_seq_length=4096,
        packing=True,                # sample packing on
        bf16=True,
        gradient_checkpointing=True,
    ),
)
trainer.train()
```

DPO / GRPO are the same shape — swap the trainer class and the
config. `accelerate launch --config_file fsdp.yaml train.py`
distributes across N GPUs without code changes.

## 5. Why it's interesting

- **The reference implementation for every preference-optimisation
  paper that matters** — DPO, IPO, KTO, ORPO, SimPO, BCO, CPO,
  Online-DPO, Nash-MD, RLOO, GRPO all land here as first-class
  trainers, usually within weeks of the paper. Pretraining and
  post-training labs read this repo as the canonical source for
  "what does the loss actually look like in code".
- **Composability with the HF stack is the design choice** —
  trainers subclass `transformers.Trainer`, accept `peft` LoRA /
  QLoRA configs, hand off to `accelerate` for distribution, log to
  `wandb` / `mlflow` / `tensorboard` / `comet` / `swanlab` /
  `trackio`. Replacing the optimiser, the data collator, or the
  loss is a subclass override, not a fork.
- **GRPO + custom reward functions = RLVR in 50 lines** — the
  DeepSeek-R1-Zero / Tülu-3 / OpenR1 reasoning-RL pattern is a
  `GRPOTrainer(reward_funcs=[my_unit_test_reward, my_format_reward])`
  call; the trainer handles rollout batching, group-relative
  advantage estimation, and KL regularisation against the reference
  model. The whole reasoning-RL ecosystem (open-r1, simpleRL-reason,
  Tülu-3, OpenRLHF wrappers) consumes this trainer.
- **Multimodal post-training is in-tree, not a side branch** —
  `SFTTrainer` and `DPOTrainer` both accept `processing_class` for
  VLMs (Llama-3.2-Vision, Qwen2-VL, Pixtral, Idefics3, SmolVLM,
  LLaVA-OneVision) so image-text SFT and image-text preference
  optimisation use the same surface as text-only.
- **Everything downstream pins to it** — Unsloth's
  `FastLanguageModel.get_peft_model(...)` returns a model the
  upstream `trl` trainers accept as-is; Axolotl's YAML resolves to
  `trl` trainer constructors; Llama-Factory wraps `trl`; the
  `huggingface/smol-course` and `huggingface/alignment-handbook`
  recipes are `trl` configs. Updating `trl` is rarely optional for
  any team doing post-training.

## 6. Caveats

- v1.x is **post-1.0 stable** but the API still evolves between
  minor versions (v1.0 → v1.3 dropped some legacy aliases, renamed
  a few config fields). Pin the version in production training
  scripts; `trl env` prints the resolved versions of `trl` /
  `transformers` / `accelerate` / `peft` / `datasets` / `torch` for
  reproducibility.
- PPO is supported but **DPO / GRPO are the recommended path** for
  most teams — PPO's value-network + reward-model + reference-model
  triple is operationally heavier than DPO's "one model + one
  reference" or GRPO's "one model + group-relative baseline".
- Distribution is **`accelerate`-shaped, not Megatron-shaped** —
  TP / PP / SP for 70B+ models works through FSDP2 + TP, but
  Megatron-style 3D parallelism for 100B+ pretraining is out of
  scope; the upstream message is "use Nanotron / Megatron-LM /
  Llama-Factory's Megatron path for that, then `trl` for the
  preference / RL stage".
- GRPO is **memory-heavy** — N rollouts per prompt × group size,
  plus the reference model in memory. Pair with vLLM
  (`use_vllm=True` in `GRPOConfig` spawns a vLLM rollout server)
  for tractable throughput on > 7B models.

## 7. Where it sits in the catalog

- **Pair with**: [`unsloth`](../unsloth/) (faster kernels, same
  trainer surface), [`axolotl`](../axolotl/) (YAML wrapper that
  resolves to `trl` trainers), [`peft`](../peft/) (LoRA / QLoRA /
  DoRA adapter implementation `trl` consumes),
  [`liger-kernel`](../liger-kernel/) (fused kernels `trl` enables
  via `use_liger_kernel=True`), [`mergekit`](../mergekit/) (merge
  the post-trained checkpoint with siblings),
  [`vllm`](../vllm/) (rollout server for GRPO / Online-DPO),
  [`lighteval`](../lighteval/) / [`lm-evaluation-harness`](../lm-evaluation-harness/)
  (evaluate the resulting checkpoint).
- **Don't use for**: pretraining from scratch (use Nanotron /
  Megatron-LM / [`litgpt`](../litgpt/)); inference serving (use
  [`vllm`](../vllm/) / [`text-generation-inference`](../text-generation-inference/)
  / [`sglang`](../sglang/) / [`lmdeploy`](../lmdeploy/));
  classical-ML supervised tasks (use plain `transformers.Trainer`).
