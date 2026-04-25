# sglang

> Snapshot date: 2026-04. Upstream: <https://github.com/sgl-project/sglang>

**High-throughput LLM serving framework with RadixAttention prefix
caching and a structured-generation frontend DSL.** SGLang ships an
OpenAI-compatible HTTP server (`python -m sglang.launch_server`) plus
a Python frontend (`@sgl.function`) that compiles control-flow
programs (forks, regex / JSON constraints, tool calls) into
batched server-side execution. The RadixAttention KV-cache reuses
shared prefixes across concurrent requests, which is what makes the
multi-turn / multi-branch / few-shot workloads cheap.

## Repo + version + license

- Repo: <https://github.com/sgl-project/sglang>
- Latest release: **`v0.5.10.post1`** (2026-04-09)
- HEAD on `main`: `0384949`
- License: **Apache-2.0** —
  <https://github.com/sgl-project/sglang/blob/main/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `main`
- Language: Python (CUDA / Triton kernels)

## Install

```bash
pip install -U "sglang[all]"
# OpenAI-compatible server
python -m sglang.launch_server \
  --model-path Qwen/Qwen3-Coder-30B-A3B-Instruct \
  --host 0.0.0.0 --port 30000
# point any OpenAI client at http://127.0.0.1:30000/v1
```

## Niche

The "**[vllm](../vllm/) alternative that wins on prefix-heavy and
structured-output workloads**" slot. vLLM still leads on raw single-
prompt throughput on most shapes; SGLang typically wins on (a) agent
loops with long shared system prompts (RadixAttention reuses the
prefix KV across the whole batch), (b) constrained / JSON / regex
decoding (compiled FSM, not token-by-token rejection), and (c)
speculative-decoding + grammar combos. Pairs with
[openllm](../openllm/) if you want a higher-level pin-the-bento
deployment surface, and with [openllmetry](../openllmetry/) for
OTel traces of the served endpoint.

## Why it matters

- RadixAttention is the differentiator: a radix tree over the KV
  cache keyed by token-prefix, so a 200-line system prompt shared
  by 500 concurrent agent calls is computed once — the published
  microbench is 2–5× higher throughput than vanilla paged-attention
  on long-shared-prefix workloads.
- The `@sgl.function` frontend lets you express forks, JSON-schema
  constraints, and tool calls as ordinary Python; the runtime
  batches and constrains decoding server-side, removing a whole
  class of client-side retry loops.
- Backend covers NVIDIA (CUDA + FlashInfer), AMD (ROCm), and
  TPU/Intel paths; quantization covers FP8 / AWQ / GPTQ / INT4 /
  W8A8 — the same `launch_server` flag set works across them.
- Mainline integration target for several recent releases (DeepSeek,
  Qwen3, Llama 4, GLM, multimodal Qwen-VL) — typically day-zero
  serving support before downstream wrappers catch up.
