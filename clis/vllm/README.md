# vllm

> Snapshot date: 2026-04. Upstream: <https://github.com/vllm-project/vllm>

"**A high-throughput and memory-efficient inference and serving
engine for LLMs.**" `vllm` is the de-facto open-source LLM
inference server for NVIDIA-class GPU boxes (and increasingly AMD
ROCm / Intel Gaudi / TPU / CPU). The two ideas it's famous for —
**PagedAttention** (KV-cache stored in fixed-size blocks like a VM
page table, eliminating fragmentation and enabling near-100 % cache
utilisation) and **continuous batching** (requests join and leave
the in-flight batch every step instead of waiting for batch-end) —
together deliver an order of magnitude more throughput at the same
latency than vanilla `transformers.generate`. The `vllm serve`
subcommand exposes the resulting engine as an OpenAI-compatible
HTTP API on `0.0.0.0:8000`, which is what every other catalog
entry's "any OpenAI-compatible endpoint" cell points at when the
self-hosting box has a real GPU.

## 1. Install footprint

- `pip install vllm` (Python ≥ 3.9, ≤ 3.12; CUDA 12.4+ wheel by
  default). Pulls `torch`, `transformers`, `tokenizers`,
  `xformers` / `flashinfer` (attention kernels), `ray`, `fastapi`,
  `uvicorn`, `outlines` (for guided decoding), `prometheus-client`.
  Heavy: a fresh install + Llama-3.1-8B weights is ~30 GB on disk.
  Hardware-specific wheels: `pip install vllm` for CUDA, separate
  wheels / Docker images for ROCm (`rocm/vllm`), Intel Gaudi
  (`vllm-fork`), CPU-only (`pip install vllm-cpu`), TPU (`vllm-tpu`).
- CLI binary: `vllm` (Typer). Subcommands: `serve`, `chat`,
  `complete`, `bench`, `run-batch`, `collect-env`. Default and
  load-bearing one is `vllm serve <model>` which boots the OpenAI-
  compatible HTTP server.
- Docker image: `vllm/vllm-openai:latest` (CUDA), runnable with
  `docker run --gpus all -v ~/.cache/huggingface:/root/.cache/huggingface
  -p 8000:8000 vllm/vllm-openai --model <hf-id>` — by far the most
  common production path.
- Configuration is CLI flags (`--tensor-parallel-size`,
  `--gpu-memory-utilization`, `--max-model-len`, `--enable-prefix-caching`,
  `--quantization awq|gptq|fp8|bitsandbytes`, `--dtype`,
  `--speculative-model`) plus an optional YAML config; full reference
  in `vllm serve --help`.

## 2. Repo + version + license

- Repo: <https://github.com/vllm-project/vllm>
- Latest release: **v0.19.1** (2026-04)
- License: **Apache-2.0** —
  <https://github.com/vllm-project/vllm/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (with C++/CUDA kernels)

## 3. Models supported

Very wide. Llama 1/2/3/3.1/3.2/3.3/4 (incl. 17B16E and other MoE
variants), Qwen 1.5 / 2 / 2.5 / 3 (Coder, VL), Mistral / Mixtral
(incl. Mixtral-8x22B), Phi 1.5 / 2 / 3 / 3.5 / 4, Gemma 1 / 2 / 3,
DeepSeek-V2 / V3 / R1 / Coder, Yi, Command-R / Command-R+,
Granite-Code, StarCoder 1/2, Falcon, Bloom, Baichuan, Aquila,
ChatGLM 2 / 3 / 4, InternLM 1 / 2, MPT, OPT, Pythia, GPT-J / NeoX,
Stable LM, Jamba (state-space hybrid), and an expanding catalog of
multimodal / VLMs (LLaVA-1.5/1.6, LLaVA-NeXT, Qwen2-VL / Qwen2.5-VL,
Phi-3-Vision, MiniCPM-V, InternVL, Pixtral, MLLama, Aria),
embedding models (`bge-*`, `e5-*`, `nomic-*`, sentence-transformers
catalog), classification / reward models (`bge-reranker-*`,
`Skywork-Reward-*`). Custom models via the
`vllm.model_executor.models` registry. Quantisation: AWQ, GPTQ,
GPTQ-Marlin, FP8 (E4M3 / E5M2), INT8, INT4, BitsAndBytes, AQLM,
Squeezeller, GGUF read (limited subset).

## 4. MCP support

No first-party MCP server / client; vLLM is a model serving layer,
not an agent host. The path is **point an MCP-aware client
([`opencode`](../opencode/), [`claude-code`](../claude-code/),
[`crush`](../crush/), [`fast-agent`](../fast-agent/),
[`goose`](../goose/), [`pydantic-ai`](../pydantic-ai/)) at the vLLM
endpoint as the OpenAI-compatible model backend** and let the
client own MCP. Tool calling works via the OpenAI tool-call schema
(`--enable-auto-tool-choice --tool-call-parser <model-family>`)
which is what MCP clients translate to under the hood.

## 5. Sub-agent model

None — vLLM is a runtime layer, not an agent. Concurrency is
**continuous batching with paged-KV-cache reuse** inside the
engine: hundreds of concurrent OpenAI HTTP requests share the GPU
without head-of-line blocking, common prompt prefixes (system
prompt, few-shot demos) are deduplicated in the KV-cache via
prefix caching, and speculative decoding (`--speculative-model
<small-draft>`) lets a small draft model propose tokens that the
big target model verifies in parallel for ~1.5–3× decode
speedup. Multi-model fleets live at the orchestration layer
([Kubernetes / vLLM Production Stack /
[`openllm`](../openllm/)]), not in the engine.

## 6. Telemetry stance

Off (no analytics endpoint baked into the engine itself; no
phone-home on serve / generate). Operational telemetry is the
Prometheus `/metrics` endpoint (request latency histograms, KV-cache
utilisation, queue depth, GPU memory, throughput) which is
**opt-in scraping** by your monitoring stack — vLLM exposes the
data, it does not push anywhere. Egress on first run is
HuggingFace Hub for weights pull (set `HF_HOME=/some/cache` to
control where they land) plus the configured tokeniser / processor
files; no further outbound calls during steady-state serving.
Server binds `0.0.0.0:8000` by default — flip to
`--host 127.0.0.1` for local-only.

## 7. Prompt-cache strategy

**Automatic prefix caching** is first-class and the load-bearing
production feature: `--enable-prefix-caching` (default `True` for
most engines as of recent releases) hashes the input token
sequence prefix-by-prefix into the paged KV-cache, so the second
request that shares a system-prompt + few-shot prefix with the
first re-uses the cached attention state instead of re-computing it
— concretely, a long system prompt of 4 k tokens shared across a
chat session amortises to near-zero on every turn after the first.
Combined with paged-KV (no fragmentation) the cache utilisation is
typically 90 %+, which is what makes vLLM's published throughput
numbers stand. Per-request `cache_salt` parameter lets you
opt-out / scope per tenant. No client-side response cache —
identical full-prompt re-runs do re-decode (use a wrapping HTTP
cache for that).

## 8. Hot keybinds

N/A — server / library, not a TUI. Operationally-useful CLI:

- `vllm serve <model> --tensor-parallel-size 2 --gpu-memory-utilization 0.9`
  — boot the server (most-common command).
- `vllm chat --model <hf-id-or-path>` — terminal chat REPL against
  a local serve, useful for smoke-testing a freshly-served model.
- `vllm bench serve --model ... --request-rate ...` — synthetic
  load-gen with TTFT / TPOT / throughput percentiles, the
  reference benchmark for "did my config regress?".
- `vllm run-batch --input-file requests.jsonl --output-file out.jsonl
  --model ...` — offline batch inference (no HTTP), the right
  shape for million-row synthetic-data jobs from
  [`distilabel`](../distilabel/).

## 9. Killer feature, weakness, when to choose

**Killer feature.** **PagedAttention + continuous batching = the
performance floor for self-hosted LLM serving on NVIDIA hardware,
behind one OpenAI-compatible URL.** Same A100 80 GB box that
serves Llama-3.1-70B at ~5 req/s with `transformers.generate`
serves it at 50–100 req/s with vLLM under realistic chat load,
because (a) the KV-cache is page-allocated so a 4 k-context
request and a 32 k-context request co-tenant the same GPU without
fragmentation, (b) requests join the in-flight batch on the next
forward pass instead of waiting for the previous batch to finish,
and (c) common prompt prefixes (system prompt, RAG header) hit the
prefix cache and skip prefill entirely. Add speculative decoding
(`--speculative-model <small-draft>`) for another 1.5–3× on
decode-bound workloads. Add tensor / pipeline / data parallelism
across N GPUs with one flag pair. Wide model support (Llama / Qwen
/ Mixtral / DeepSeek / Phi / Gemma / multimodal / embedding /
reranker) and wide quantisation support (AWQ / GPTQ / FP8 / INT8 /
BitsAndBytes) means almost any open-weights model on HuggingFace
boots under `vllm serve <hf-id>` with no model-specific glue. The
result is **the OpenAI-compatible URL the rest of this catalog
points at when the self-hosting box has real GPUs** — the
production-shaped peer of [`mlx-lm`](../mlx-lm/) on the Apple-
Silicon side and [`llamafile`](../llamafile/) on the
"single-file demo" side.

**Weakness.** **NVIDIA / Linux is the default-supported tier;
everything else is an asterisk.** ROCm support exists and is
active but lags CUDA on model coverage and kernel maturity; Intel
Gaudi / TPU paths are vendor forks; CPU-only is technically
possible (`vllm-cpu`) but typically 10–50× slower than the same
model under [`llama.cpp`](../llamafile/) on the same CPU because
vLLM's optimisation surface assumes GPU memory characteristics.
Apple Silicon has no first-class path — use [`mlx-lm`](../mlx-lm/)
or [`llamafile`](../llamafile/) on Mac. Cold-start cost is real:
the first request after `vllm serve <large-model>` boots can
take 30–120 s for weight load + cache warmup, which is fine for
long-running services but awkward for serverless / scale-to-zero.
Configuration surface is wide (`--tensor-parallel-size`,
`--pipeline-parallel-size`, `--max-model-len`,
`--gpu-memory-utilization`, `--quantization`, `--kv-cache-dtype`,
`--swap-space`, `--block-size`, `--num-scheduler-steps`,
`--enable-chunked-prefill`, `--max-num-batched-tokens`) — the
defaults are usually right, but the failure mode of a wrong combo
is "OOM at request 47" or "throughput is half what the docs
promised", and diagnosing requires reading the engine logs +
`nvidia-smi`. The release cadence is fast (multiple minor versions
per month) which is great for new model support and bad if you
need a stable target — pin a specific version in production and
upgrade deliberately.

**When to choose.** You have one or more NVIDIA GPUs (or an AMD
ROCm or Intel Gaudi box) and you want to serve an open-weights
LLM as an OpenAI-compatible endpoint at production throughput,
and you would rather operate one well-understood serving binary
than stitch together TGI / TensorRT-LLM / SGLang / Triton /
custom code yourself. Pair with [`openllm`](../openllm/) on top
if you want a *bento manifest* abstraction (per-model dtype + chat
template + GPU shape pinned in YAML, reproducible from laptop to
cluster) — `openllm` picks `vllm` as the backend automatically
for most catalog models. Pair with [`outlines`](../outlines/) to
get token-level constraint masking (vLLM exposes the logits-
processor hook `outlines` plugs into) for guaranteed-valid
structured output behind the same endpoint. Pair with
[`distilabel`](../distilabel/) for million-row synthetic-data
jobs that benefit from `vllm`'s in-process batched generation.
Skip in favour of [`mlx-lm`](../mlx-lm/) if you are on Apple
Silicon; skip in favour of [`llama.cpp`](../llamafile/) if your
hardware is CPU-only or single-laptop-GPU and the throughput floor
is fine; skip in favour of [`sglang`](https://github.com/sgl-project/sglang)
if your workload is heavy on structured / multi-turn / RAG
patterns where SGLang's RadixAttention beats vLLM's prefix cache
for your specific access pattern (benchmark both before
committing).
