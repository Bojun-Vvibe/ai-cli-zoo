# mlx-lm

> Snapshot date: 2026-04. Upstream: <https://github.com/ml-explore/mlx-lm>

"**Run LLMs with MLX.**" `mlx-lm` is the canonical Apple-Silicon LLM
runtime: a Python package that loads any MLX-converted Hugging Face
model into unified memory on M-series Macs and exposes a family of
small, sharply-scoped CLIs — `mlx_lm.generate` (one-shot prompt),
`mlx_lm.chat` (REPL with persistent context), `mlx_lm.server` (an
OpenAI-compatible HTTP endpoint at `127.0.0.1:8080` that any other
CLI in this catalog can target), `mlx_lm.lora` (LoRA / QLoRA
fine-tune), `mlx_lm.fuse` (merge an adapter back into base weights),
and `mlx_lm.convert` / `mlx_lm.upload` (quantize a HF repo to 2/3/4/6/
8-bit and push it to the `mlx-community` org). The package backs the
4900-model `mlx-community` Hugging Face org and is the Apple-blessed
inference layer most other Mac-local LLM tools end up running on.

## 1. Install footprint

- `pip install mlx-lm` (pure Python wheel; pulls `mlx`, `mlx-metal`,
  `transformers`, `huggingface-hub`, `numpy`, `protobuf`); also
  `conda install -c conda-forge mlx-lm`.
- Apple Silicon (M1 / M2 / M3 / M4) only — `mlx` requires Metal.
  macOS ≥ 13.5, Python ≥ 3.9.
- CLI surface is a family of namespaced entrypoints:
  `mlx_lm.generate`, `mlx_lm.chat`, `mlx_lm.server`, `mlx_lm.lora`,
  `mlx_lm.fuse`, `mlx_lm.convert`, `mlx_lm.upload`,
  `mlx_lm.cache_prompt`, `mlx_lm.evaluate`, `mlx_lm.distributed_*`.
- Default model on first run: `mlx-community/Llama-3.2-3B-Instruct-4bit`
  (~1.8 GB), pulled lazily into `~/.cache/huggingface/hub/`.

## 2. Repo + version + license

- Repo: <https://github.com/ml-explore/mlx-lm>
- Latest release: **v0.31.3** (2026-04-22)
- License: **MIT** —
  <https://github.com/ml-explore/mlx-lm/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (with MLX / Metal kernels under the hood)

## 3. Models supported

Anything in MLX format. The `mlx-community` HF org publishes ready-to-
run conversions of Llama 3.x / 4.x, Qwen 2.5 / 3 (incl. Qwen3-Coder,
Qwen3-VL), Mistral, Mixtral, Phi-3 / 4, Gemma 2 / 3, DeepSeek-V2 / V3
/ R1, Yi, Command-R, GLM-4, Granite, OLMo, StableLM, MiniCPM, Hermes,
plus VLMs (Llava, Qwen2-VL, MiniCPM-V) — typically within days of
upstream release. Custom models work via `mlx_lm.convert
--hf-path <hf-repo> --quantize --q-bits 4`. Quantization options:
2 / 3 / 4 / 6 / 8-bit, group-quant, AWQ, GPTQ-imported. Distributed
inference across multiple Macs via `mlx.distributed` for models that
don't fit on one machine.

## 4. MCP support

No — `mlx-lm` is a model runtime, not an agent. The way to combine it
with MCP is to run `mlx_lm.server --port 8080` and point an MCP-aware
client ([`opencode`](../opencode/), [`codex`](../codex/),
[`claude-code`](../claude-code/), [`continue`](../continue/),
[`fast-agent`](../fast-agent/)) at it as an OpenAI-compatible model
backend; that client then provides MCP. The server speaks the
`/v1/chat/completions`, `/v1/completions`, and `/v1/models` subset of
the OpenAI API plus streaming.

## 5. Sub-agent model

None. `mlx-lm` is one process per loaded model; concurrency is
request-level inside `mlx_lm.server` (continuous batching with KV-
cache reuse). Fan-out / orchestration belongs to whatever agent
framework you put in front of it.

## 6. Telemetry stance

Off. No analytics, no opt-in pings, no usage reporting. The only
egress is Hugging Face Hub on first model pull (and only if the
weights aren't already cached); after that the runtime is fully
offline. `mlx_lm.server` binds to `127.0.0.1` by default; pass
`--host 0.0.0.0` only if you actually want LAN exposure.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Apple Silicon's unified memory turned into a
production LLM endpoint with one command.** `mlx_lm.server --model
mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit` on a 64 GB M3 Max
gets you ~80 tok/s of a coding-grade model behind an OpenAI-compatible
URL — no GPU box, no Docker, no quantization toolchain to assemble.
The CLI family is small and orthogonal: each binary does one thing
(generate / chat / serve / convert / fine-tune) with consistent
flags, so you can shell-script the whole weights-to-endpoint pipeline
in fifteen lines. LoRA fine-tuning on a 7B model fits in 16 GB of
unified memory and uses the same data path. Compared to
[`llama.cpp`](../llamafile/) on a Mac, MLX's Metal kernels are 1.5–
3× faster on most decode workloads because they're written for
Apple's shader model rather than abstracted across CUDA / Metal /
ROCm.

**Weakness.** **Apple Silicon only, and Apple-Silicon-shaped.**
There is no x86 / NVIDIA / AMD path — if your team has even one
non-Mac dev box you'll need a separate runtime for them
([`ollama`](../ollama/), [`vllm`](#) via [`openllm`](../openllm/), or
[`llamafile`](../llamafile/)). Quantization formats are MLX-specific:
existing GGUF / AWQ / GPTQ weights need a `mlx_lm.convert` pass
before they load (the `mlx-community` HF org has done this for the
popular models, but a fresh upstream release lags by hours-to-days).
The CLI namespace `mlx_lm.*` is unfamiliar to anyone used to a
single-binary tool — tab-completion is rough, and the underscore-
versus-hyphen mismatch (the *package* is `mlx-lm`, the *commands*
are `mlx_lm.*`) trips first-time users. No built-in chat templating
override UI — if a model ships a broken Jinja template, you patch
the tokenizer config by hand.

**When to choose.** You have a Mac with ≥ 16 GB unified memory
(ideally ≥ 36 GB), you want a *local* model behind an OpenAI-
compatible endpoint that the rest of this catalog can target, and
you want the fastest decode the hardware can do. Pair with
[`opencode`](../opencode/) / [`codex`](../codex/) /
[`continue`](../continue/) / [`aider`](../aider/) /
[`mods`](../mods/) — point any of them at `http://127.0.0.1:8080/v1`
with model name `mlx-community/<repo>` and the entire stack runs
without leaving the laptop. Use [`mlx_lm.lora`](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/LORA.md)
when you want a domain-tuned variant from a few thousand examples
overnight on the same machine. Skip in favour of
[`openllm`](../openllm/) or `vllm` if you're targeting NVIDIA GPUs
in a cloud / Kubernetes deployment, or [`ollama`](../ollama/) if you
need a single binary that also runs on Linux / Windows boxes for the
rest of your team.
