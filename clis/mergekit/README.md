# mergekit

> Snapshot date: 2026-04. Upstream: <https://github.com/arcee-ai/mergekit>
> License file: <https://github.com/arcee-ai/mergekit/blob/main/LICENSE>
> Pinned: `v0.1.4` (2025-10-31). Default branch `main` continues to receive
> commits after the last tagged release; pin the SHA in production training
> pipelines rather than tracking `latest`.

A Python CLI + library that **merges the weights of multiple pretrained
language models into a single model** without any further training. Same
shape as the well-known recipe surface around model souping and
TIES/SLERP/DARE/Linear/Task-Arithmetic, but packaged as a single
declarative `mergekit-yaml` runner: you write a YAML recipe naming the
base model, the donor models, the merge method, and per-layer / per-tensor
weights, and the CLI streams the layered tensors through the merge math
and writes a fresh HuggingFace checkpoint to disk.

There is no agent, no training loop, no inference engine — it is a
deterministic weight-arithmetic tool that takes N safetensors directories
in and emits one safetensors directory out, ready to be served by
[`vllm`](../vllm/), [`llama.cpp`](../llama.cpp/), or
[`openllm`](../openllm/).

## 1. Install footprint

- `pip install mergekit` (PyPI). Pulls in `torch`, `transformers`,
  `safetensors`, `accelerate`, `huggingface-hub`, `pydantic`, `click`.
  Also installable from source via `pip install -e .` for the
  experimental MoE / evolve extras.
- Disk: the binary itself is small, but you need room for **base + all
  donor checkpoints + output**. A 70B 5-way merge over fp16 weights
  needs ~700 GB of free disk plus enough RAM (or `--lazy-unpickle` +
  `--out-shard-size`) to stream tensors layer by layer.
- GPU is **not required for the merge itself** — most methods are pure
  weight arithmetic and run on CPU. GPU only enters if you opt into
  `mergekit-evolve` (see §9), which evaluates candidate merges with a
  real eval harness.
- Output is a standard HuggingFace model directory: `config.json`,
  `tokenizer.*`, `model.safetensors.index.json`, sharded
  `model-XXXXX-of-YYYYY.safetensors`. Drop straight into
  `vllm serve <dir>` or `huggingface-hub upload`.

## 2. License

LGPL-3.0.

That choice is unusual for the model-tooling layer of this catalog
(`vllm`, `litgpt`, `axolotl` are Apache-2.0). It does not affect the
*output* model — merged weights inherit the licenses of the input
models, not mergekit's — but it does affect what you can statically
link the mergekit Python package into. For most users (CLI invocation,
YAML recipes) this is a non-issue.

## 3. Models supported

Anything whose architecture mergekit knows the layer layout of. As of
0.1.4 the supported architecture list covers the bulk of the
open-weight ecosystem:

- **Llama family** — Llama 2 / 3 / 3.1 / 3.2 / 3.3 / 4 (incl. MoE
  variants where the experts share architecture)
- **Mistral / Mixtral** — including the 8x7B and 8x22B sparse MoE
  configurations (Mixtral merges have a dedicated `mergekit-moe`
  variant)
- **Qwen 1.5 / 2 / 2.5 / 3** (dense and MoE)
- **Gemma 1 / 2 / 3**
- **Phi 1.5 / 2 / 3 / 3.5 / 4**
- **DeepSeek V2 / V3 / R1 distilled dense variants**
- **Yi, InternLM, Falcon, MPT, GPT-NeoX, GPT-J, OPT, Pythia,
  StableLM, ChatGLM, Command-R(+), Granite, Solar**

The merge method catalog (`merge_method:` in the recipe) covers the
literature: `linear`, `slerp`, `task_arithmetic`, `ties`, `dare_ties`,
`dare_linear`, `breadcrumbs`, `breadcrumbs_ties`, `model_stock`,
`della`, `della_linear`, `nuslerp`, `sce`, `passthrough` (the
"frankenmerge" mode that stacks layers from different donors instead
of averaging them), plus `mergekit-moe` for assembling a sparse MoE
out of several dense fine-tunes.

## 4. MCP support

**No.** mergekit is offline weight math; there is no agent surface to
expose. If you want an agent to *trigger* merges, wrap the
`mergekit-yaml` invocation in a tiny MCP server yourself.

## 5. Sub-agent model

None. The tool is a single-process tensor pipeline: load base, walk
the layer / tensor graph, for each tensor pull the corresponding
tensors from each donor, apply the chosen merge method, write the
result. `mergekit-evolve` is the closest thing to a loop — it runs an
evolutionary search over merge recipes, evaluating each candidate
with `lm-evaluation-harness` — but it is still one process driving
deterministic eval calls, not an agentic planner.

## 6. Telemetry stance

**Off, no opt-in.** No analytics SDK in the package. The only outbound
network calls are HuggingFace Hub downloads of the base + donor
checkpoints (`HF_HUB_OFFLINE=1` plus pre-downloaded weights gives you
fully air-gapped operation) and, if you opt into `huggingface-cli
upload` at the end, the upload of the merged output.

## 7. Prompt-cache strategy

N/A. mergekit produces models, it does not run inference. The merged
output's prompt-cache behavior is whatever the serving layer
([`vllm`](../vllm/), [`llama.cpp`](../llama.cpp/), Ollama, etc.)
provides for that architecture.

## 8. Hot keybinds

No TUI. The CLI surface is a small set of click commands:

- `mergekit-yaml recipe.yml ./output --copy-tokenizer --allow-crimes`
  — the workhorse: read a YAML recipe, write a merged model directory.
  `--copy-tokenizer` brings the base's tokenizer over,
  `--allow-crimes` permits cross-architecture / mismatched-shape
  merges (use with care; many will produce broken models).
- `mergekit-yaml recipe.yml ./output --lazy-unpickle --out-shard-size
  5B` — stream tensors instead of loading everything into RAM; mandatory
  on 70B+ merges on a single workstation.
- `mergekit-mega recipe.yml ./output` — the multi-stage variant: a
  recipe can reference *another recipe* as one of its donors, and
  `mega` resolves the DAG in dependency order, materialising
  intermediate merges to a scratch directory.
- `mergekit-moe moe-recipe.yml ./output` — assemble a sparse MoE from
  several dense fine-tunes: you specify the experts, the gate
  initialisation strategy (`hidden`, `random`, `cheap_embed`, etc.),
  and the routing topology.
- `mergekit-extract-lora <merged-model> <base-model> ./lora-out
  --rank 32` — the inverse operation: factor the *delta* between a
  fine-tune (or merge) and its base into a LoRA adapter, useful for
  publishing a 200 MB adapter instead of a 140 GB full checkpoint.
- `mergekit-evolve evolve-config.yml --vllm` — evolutionary search
  over merge recipes, scored by `lm-evaluation-harness` tasks; needs
  GPU, takes hours to days.

A minimal recipe (TIES merge of two Llama-3.1-8B fine-tunes onto the
base):

```yaml
models:
  - model: meta-llama/Meta-Llama-3.1-8B-Instruct
  - model: NousResearch/Hermes-3-Llama-3.1-8B
    parameters: { weight: 0.5, density: 0.5 }
  - model: Sao10K/L3.1-8B-Niitama-v1.1
    parameters: { weight: 0.5, density: 0.5 }
merge_method: ties
base_model: meta-llama/Meta-Llama-3.1-8B
parameters: { normalize: true, int8_mask: true }
dtype: bfloat16
```

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Declarative weight arithmetic with a real
recipe registry.** mergekit's YAML format has become the de-facto
exchange format for model merging on HuggingFace — a substantial
fraction of the top-trending OSS models on the leaderboard ship their
mergekit recipe in the model card, which means you can reproduce
SOTA-on-niche-evals merges with one CLI call against weights you
already have. The TIES / DARE / breadcrumbs / model-stock catalog
covers the active research literature, `passthrough` enables the
"frankenmerge" pattern (stacking layers from different donors to grow
a model in depth without retraining) which produced multiple
top-of-leaderboard models in 2024-2025, and `mergekit-evolve` closes
the loop by letting you search the merge-recipe space against your
own eval suite. None of the other model-tooling entries in this
catalog ([`vllm`](../vllm/), [`litgpt`](../litgpt/),
[`unsloth`](../unsloth/), [`mlx-lm`](../mlx-lm/)) attempt this — they
serve, fine-tune, or quantize an existing checkpoint; mergekit is the
only one that *creates* a new one without gradient descent.

**Weakness.** Merging is **necromancy, not engineering** — there is
no theoretical guarantee that a TIES merge of two strong fine-tunes
will be stronger than either, and many recipes in the wild are the
output of someone running `mergekit-evolve` for 48 hours and
publishing whatever scored highest on a narrow eval. Quality is
extremely sensitive to per-layer / per-tensor weight choices, and the
failure mode is silent: you get a fluent model that has lost a
specific capability (math, code, multilingual) that neither donor had
lost. Disk requirements scale with model count, not with model size:
a 5-way 70B merge can need a TB of free space. `--allow-crimes`
cross-arch merges are a footgun; LGPL-3.0 is heavier than the
Apache-2.0 norm in the model-tooling layer.

**When to choose.**

- You want to **reproduce a specific HuggingFace merge** whose
  recipe is published in the model card — `mergekit-yaml` is the only
  tool that runs that recipe verbatim.
- You have **two or more fine-tunes of the same base model** that
  each excel on a different axis (instruction-following vs. code vs.
  long-context vs. multilingual) and you want a single deployment
  artifact that retains as much of each capability as possible.
- You want to **publish a 200 MB LoRA adapter** instead of a full
  fine-tuned checkpoint — `mergekit-extract-lora` factors the delta
  cleanly.
- You want to **assemble a sparse MoE** from several dense
  fine-tunes (e.g. four Llama-3.1-8B specialists into a 4x8B
  expert-routed model) without paying for from-scratch MoE training —
  `mergekit-moe` is the only OSS tool with this surface.
- You are running an **evolutionary search over merge recipes** as
  part of a model-development pipeline — `mergekit-evolve` plugs
  directly into [`lm-evaluation-harness`](../lm-evaluation-harness/).

**When not to choose.** You want to *fine-tune* a model — pick
[`axolotl`](../axolotl/), [`unsloth`](../unsloth/),
[`litgpt`](../litgpt/), or HuggingFace TRL. You want to *quantize* an
existing model — pick [`llama.cpp`](../llama.cpp/)'s `llama-quantize`
or [`mlx-lm`](../mlx-lm/)'s `mlx_lm.convert`. You want a hosted /
managed merge service — that is Arcee's commercial cloud product, not
the OSS CLI. You need a *theoretical guarantee* that the output is
better than its inputs — no merge tool can give you that; you need an
eval harness in the loop, which is what `mergekit-evolve` is for.
