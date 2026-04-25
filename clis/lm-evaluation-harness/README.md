# lm-evaluation-harness

> Snapshot date: 2026-04. Upstream: <https://github.com/EleutherAI/lm-evaluation-harness>

**The de-facto academic benchmark harness for LLM evaluation.**
EleutherAI's `lm_eval` is the harness behind the Hugging Face Open
LLM Leaderboard, the BigScience papers, and most "we ran our model
on MMLU / HellaSwag / ARC / TruthfulQA / GSM8K" numbers you see in
arXiv. A `Task` declares a `dataset_path`, a `doc_to_text` template,
a `doc_to_target`, a metric (`acc` / `acc_norm` / `f1` /
`exact_match` / `bleu` / `pass@k` / custom), and a few-shot policy;
`lm_eval` runs it across one or more model backends and emits a
deterministic JSON results blob you can publish next to the model
card.

## Repo + version + license

- Repo: <https://github.com/EleutherAI/lm-evaluation-harness>
- Latest release: **`v0.4.11`** (2026-02-13)
- HEAD on `main`: `c1c4bea` (2026-04-08)
- License: **MIT** —
  <https://github.com/EleutherAI/lm-evaluation-harness/blob/main/LICENSE.md>
- License path in repo: `LICENSE.md`
- License SPDX: `MIT`
- Default branch: `main`
- Language: Python

## Install

```bash
pip install lm-eval
# or with extras for HF / vLLM / OpenAI / Anthropic backends
pip install "lm-eval[vllm,openai,anthropic]"

# Run MMLU (5-shot) against a local HF model
lm_eval --model hf \
        --model_args pretrained=meta-llama/Llama-3.1-8B-Instruct \
        --tasks mmlu \
        --num_fewshot 5 \
        --batch_size 8 \
        --output_path results/

# Run GSM8K against an OpenAI-compatible endpoint (vLLM, Ollama, etc.)
lm_eval --model local-completions \
        --model_args base_url=http://localhost:8000/v1/completions,model=qwen2.5-7b \
        --tasks gsm8k --num_fewshot 8
```

## Niche

Reproducible academic-style benchmark runner with **300+ built-in
tasks** (MMLU, MMLU-Pro, HellaSwag, ARC-Challenge, ARC-Easy,
TruthfulQA, WinoGrande, GSM8K, MATH, HumanEval, MBPP, BBH, AGIEval,
AIME, GPQA, MT-Bench-style, multilingual subsets, BIG-Bench, plus
hundreds of community-contributed YAML tasks). Sits next to
[inspect-ai](../inspect-ai/), [deepeval](../deepeval/),
[ragas](../ragas/), [lighteval](../lighteval/), [helm](../helm/),
[promptfoo](../promptfoo/), [trulens](../trulens/), but its slot is
*"give me a number that compares to the leaderboard"*, not
"score my RAG pipeline" or "run a frontier-model safety eval".

## Why it matters

- **The reference implementation behind public leaderboards** — the
  Hugging Face Open LLM Leaderboard v1 / v2, the EleutherAI Pythia
  / GPT-NeoX papers, and most academic LLM publications since 2022
  cite `lm_eval` task configs. If you publish a number against MMLU
  or HellaSwag and *don't* use this harness, reviewers will ask why.
- **Backend matrix is broader than agent-focused harnesses** —
  HuggingFace `transformers` (CPU / CUDA / MPS / Intel XPU), vLLM,
  llama.cpp via `gguf`, OpenAI / Anthropic / Cohere / Mamba /
  TextSynth / NeuronX, plus any OpenAI-compatible HTTP endpoint
  through `local-completions` / `local-chat-completions`; the same
  task YAML runs on a 7B GGUF on a laptop and a 405B model on
  hosted infra.
- **Tasks are YAML, not Python** — `tasks/<benchmark>/<task>.yaml`
  declares `dataset_path` (HF hub or local), `doc_to_text` /
  `doc_to_target` (Jinja2), `metric_list`, `num_fewshot`,
  `description`, generation kwargs; community PRs adding GPQA,
  AIME-2024, IFEval, BBH-Hard variants don't touch Python.
- **Deterministic results JSON** — every run emits
  `results.json` + per-sample `samples_<task>.jsonl` keyed by
  `model_args` + git SHA + task version; pin all three and the
  numbers are bit-reproducible across machines (within fp tolerance
  for the model backend).
- Closest catalog peer is [inspect-ai](../inspect-ai/), but
  `inspect-ai` is the safety / agentic-eval harness (Solver chains,
  per-sample Docker sandboxes, transcript replay UI) while
  `lm-evaluation-harness` is the *academic-benchmark scoreboard*
  harness (multiple-choice loglikelihood scoring, fixed few-shot
  prompts, leaderboard parity). Use both: `lm_eval` for "how does
  the base model compare to Llama-3-70B on MMLU", `inspect` for
  "does the agent solve SWE-bench-Verified".
- MIT-licensed, no telemetry, no hosted-control-plane requirement;
  `--output_path` writes JSON locally, you publish it however you
  want.
