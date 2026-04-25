# lighteval

> Snapshot date: 2026-04. Upstream: <https://github.com/huggingface/lighteval>

**Hugging Face's lightweight LLM eval harness, accelerate-native.**
`lighteval` is HF's in-house eval framework — a leaner, more
opinionated cousin of `lm-evaluation-harness` written from scratch
to use `accelerate` as the distribution primitive (multi-GPU,
DeepSpeed, FSDP, TP) and to support the eval suites the HF model
team actually runs against new releases. Tasks are Python or YAML,
backends are HF / vLLM / TGI / OpenAI-compatible / Inference
Endpoints / Nanotron, and results land as a `Details` parquet you
can sliced-and-dice in `datasets`.

## Repo + version + license

- Repo: <https://github.com/huggingface/lighteval>
- Latest release: **`v0.13.0`** (2025-11-24)
- HEAD on `main`: `10b9104` (2026-04-15)
- License: **MIT** —
  <https://github.com/huggingface/lighteval/blob/main/LICENSE>
- License path in repo: `LICENSE`
- License SPDX: `MIT`
- Default branch: `main`
- Language: Python

## Install

```bash
pip install lighteval
# extras for the backend you want
pip install "lighteval[accelerate]"   # multi-GPU HF
pip install "lighteval[vllm]"          # vLLM
pip install "lighteval[tgi]"           # Text Generation Inference
pip install "lighteval[nanotron]"      # HF's pretraining stack

# Run a community-curated MMLU + HellaSwag + ARC suite on a HF model
lighteval accelerate \
    "model_name=meta-llama/Llama-3.1-8B-Instruct" \
    "leaderboard|mmlu|5|0,leaderboard|hellaswag|10|0,leaderboard|arc:challenge|25|0"

# Same suite via vLLM for ~5x throughput on a single H100
lighteval vllm \
    "model_name=meta-llama/Llama-3.1-8B-Instruct,gpu_memory_utilization=0.9" \
    "leaderboard|mmlu|5|0"
```

## Niche

Hugging-Face-blessed eval harness optimised for **fast multi-GPU
runs against transformer-shaped models you control**. Sits next to
[lm-evaluation-harness](../lm-evaluation-harness/),
[inspect-ai](../inspect-ai/), [helm](../helm/), [deepeval](../deepeval/),
[ragas](../ragas/), [promptfoo](../promptfoo/), but its slot is
*"my pretraining run finished, evaluate every checkpoint on the
leaderboard suites in <1 GPU-hour"*, not "score a RAG pipeline" or
"run an agentic eval".

## Why it matters

- **Accelerate-native distribution** — pass an
  `accelerate` config and `lighteval` shards the eval across GPUs
  with DeepSpeed ZeRO / FSDP / TP automatically; running MMLU on a
  70B model across 8×H100 is one command, no manual
  `torch.distributed.run` boilerplate. `lm-evaluation-harness`
  supports multi-GPU but the path is rougher.
- **vLLM + TGI + Nanotron paths are first-class** — `lighteval
  vllm`, `lighteval tgi`, `lighteval nanotron` are top-level
  subcommands, not opt-in extras; the HF training stack
  (Nanotron) can be evaluated *during* training without exporting
  weights.
- **Built around HF's own eval recipes** — the `community/` task
  registry mirrors the suites HF announces models on
  (`leaderboard|*` for OpenLLM-Leaderboard parity,
  `lighteval|*` for HF's own picks like IFEval, GPQA-Diamond,
  MUSR, BBH-CoT, MATH-Lvl-5, MMLU-Pro). Reproducing a HF model
  card's claimed numbers is closer to "click run" than with most
  other harnesses.
- **Details parquet, not just `results.json`** — every prediction +
  reference + score lands in `details_<task>.parquet`, queryable
  with `datasets.load_dataset("parquet", ...)` for "which 50 MMLU
  questions did the model regress on between checkpoint 12k and
  checkpoint 24k". The dataset-native output format is the design
  difference vs. [lm-evaluation-harness](../lm-evaluation-harness/)'s
  JSONL.
- **Pick `lighteval` over [`lm-evaluation-harness`](../lm-evaluation-harness/)**
  when (a) you're inside the HF ecosystem already (HF Hub model,
  `accelerate` config, `datasets` tooling) and (b) leaderboard
  parity + multi-GPU throughput matter more than the larger 300+
  task catalog. Pick `lm-evaluation-harness` when you need long-tail
  / community-contributed tasks (AIME-2024 variants, BBH-Hard
  splits, niche multilingual benchmarks) or backends beyond HF / vLLM
  / TGI.
- **Pick `lighteval` over [`inspect-ai`](../inspect-ai/) /
  [`deepeval`](../deepeval/)** when the system-under-test is a
  *base / instruct model on benchmark scoring*, not an agent or RAG
  pipeline. Multiple-choice loglikelihood scoring + fixed few-shot
  prompts is the design point; tool-use sandboxes are not.
- MIT-licensed, no telemetry, results are local files; the trust
  profile matches `lm-evaluation-harness` and `inspect-ai`.
