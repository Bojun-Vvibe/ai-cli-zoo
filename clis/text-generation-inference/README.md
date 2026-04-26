# text-generation-inference

> Snapshot date: 2026-04. Upstream:
> <https://github.com/huggingface/text-generation-inference>
> License file: <https://github.com/huggingface/text-generation-inference/blob/main/LICENSE>
> (Apache-2.0).
> Pinned: `v3.3.7` (latest tagged release).

Hugging Face's **production-grade open-weight inference server** —
"TGI" — a Rust router + Python sharded model server that takes a
Hugging Face Hub model ID and exposes it as a streaming HTTP +
gRPC endpoint with paged-attention KV cache, continuous batching,
tensor parallelism across GPUs, speculative decoding, and an
OpenAI-compatible `/v1/chat/completions` route. Sits in the same
neighborhood as [`vllm`](../vllm/), [`sglang`](../sglang/), and
[`lorax`](../lorax/) but with HF-specific niceties (Hub auth, safe
defaults for Llama / Mistral / Mixtral / Qwen / Gemma / Phi,
Tokenizers integration, `Inference Endpoints` parity).

## 1. Install footprint

- **Docker (the default path)**:
  `docker run --gpus all --shm-size 1g -p 8080:80 -v
  $PWD/data:/data ghcr.io/huggingface/text-generation-inference:3.3.7
  --model-id meta-llama/Llama-3.1-8B-Instruct`. Image is ~8 GB
  (Rust router + Python server + CUDA + flash-attention + the
  paged-attention kernels).
- **Native install (Linux + CUDA)**: `make install` from the repo;
  pulls Rust toolchain (`cargo`), Python ≥ 3.9, PyTorch with CUDA,
  `text-generation-server` Python package, plus pre-built
  flash-attention / paged-attention wheels. Not recommended outside
  development — the Docker image bakes the kernel build matrix.
- **CLI surface**:
  - `text-generation-launcher` — the Rust router binary; the
    primary entry point. `--model-id`, `--num-shard`,
    `--quantize bitsandbytes-nf4|gptq|awq|eetq|fp8`,
    `--max-input-tokens`, `--max-total-tokens`,
    `--speculate N` (Medusa / EAGLE), `--lora-adapters`.
  - `text-generation-server` — the Python per-shard worker; the
    router fans out to N of these.
  - `text-generation-benchmark` — load-tests an endpoint with
    configurable concurrency and prompt distributions.
- **Hardware**: NVIDIA (CUDA), AMD ROCm, Intel Gaudi (HPU), AWS
  Inferentia2 (Neuron), Google TPU. CPU mode exists but is
  development-only.

## 2. Repo + version + license

- Repo: <https://github.com/huggingface/text-generation-inference>
- Latest release: **v3.3.7**
- License: **Apache-2.0** —
  <https://github.com/huggingface/text-generation-inference/blob/main/LICENSE>
- Default branch: `main`
- Languages: Rust (router), Python (server / kernels glue)

## 3. Models supported

Open-weight causal LMs from the Hugging Face Hub: Llama 1 / 2 / 3 /
3.1 / 3.2 / 3.3, Mistral 7B / Nemo / Small / Large, Mixtral 8x7B /
8x22B, Qwen 1.5 / 2 / 2.5 / 3, Gemma 1 / 2 / 3, Phi-2 / 3 / 4,
Falcon, MPT, StarCoder / StarCoder2, CodeLlama, DeepSeek-V2 / V3 /
R1 distilled, Yi, Granite, Command-R / R+, OLMo, Mamba / Jamba
(SSM + hybrid). Vision-language models: LLaVA-NeXT, Idefics 2 / 3,
PaliGemma. Embedding models served via the sibling
`text-embeddings-inference` project (same routing pattern, separate
repo). Quantization: bitsandbytes (int8 / nf4), GPTQ (3 / 4 / 8-bit),
AWQ, EETQ, FP8 (H100 / MI300). Multi-LoRA: load N adapters at boot
and route per-request via the `adapter_id` field — same path
[`lorax`](../lorax/) productionalizes, here as a built-in.

## 4. MCP support

No (TGI is an inference server, not an agent runtime). The
OpenAI-compatible `/v1/chat/completions` surface means any
MCP-speaking agent ([`claude-code`](../claude-code/),
[`opencode`](../opencode/), [`cline`](../cline/),
[`aider`](../aider/), [`continue`](../continue/)) can target a TGI
endpoint as their model provider — TGI hosts the model, the agent
hosts the MCP loop.

## 5. Sub-agent model

None — TGI is a single-model serving layer. Tool-calling support is
exposed as **structured output**: the router accepts a `tools=[...]`
array on `/v1/chat/completions` plus
`response_format={"type": "json_schema", ...}`, applies grammar-
constrained decoding (Outlines-style) so the chosen model emits a
syntactically valid tool call, and returns it in OpenAI's standard
`message.tool_calls` shape. Agentic loops live in the client.

## 6. Telemetry stance

Off by default. Native **OpenTelemetry** export is built in: set
`OTLP_ENDPOINT` and the router emits per-request spans (queue time,
prefill time, decode time, batch size at each step) plus Prometheus
metrics on `:80/metrics`. No vendor analytics; egress is whatever
OTel collector and Prometheus you point it at. Hub access for the
initial weight download is the only outbound network call at boot
(set `HF_HUB_OFFLINE=1` after the weights are cached for fully
air-gapped operation).

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Continuous batching + paged-attention with
multi-LoRA + speculative decoding, all behind a single
OpenAI-compatible URL, and an OpenTelemetry trace per request that
tells you exactly where the latency went.** The router admits
incoming requests into the running batch step-by-step instead of
waiting for the current batch to finish (continuous batching), the
KV cache is paged so long contexts and short contexts can share GPU
memory without fragmentation (paged-attention), and a single server
can host a base model plus dozens of LoRA adapters and route per
request — turning "30 customer-tuned variants of Llama-3-8B" from a
30-replica problem into a 1-replica problem. Speculative decoding
(`--speculate N` with Medusa / EAGLE / draft-model) shaves 1.5-3×
off decode latency for long completions on the right hardware. The
`text-generation-benchmark` binary in the same repo lets you
characterize the throughput / latency / VRAM curve for *your*
prompts on *your* GPU before you commit to a SKU.

**Weakness.** **It's a serving layer, not a workflow.** TGI
doesn't fine-tune, doesn't manage prompts, doesn't store traces of
your business logic, doesn't help you write the agent. You bring
the model, the agent loop, the eval harness, and the deployment
plumbing; TGI just makes the GPU efficient. The Docker image is
heavy (~8 GB) because the CUDA + flash-attention + paged-attention
kernel matrix is baked in — fine in production, painful for laptop
poking (use [`ollama`](../ollama/) or [`llamafile`](../llamafile/)
there). Configuration cliff: 60+ launcher flags, and the
default-good cases (Llama-family on a single A100 / H100) are very
different from the edge cases (multi-node tensor + pipeline parallel
on consumer GPUs), where you will be reading source. Competing
Apache-2.0 stacks ([`vllm`](../vllm/), [`sglang`](../sglang/)) move
faster on bleeding-edge model architectures; TGI prioritizes
stability + Hub integration over week-one support for new model
families.

**When to choose.** You are **deploying open-weight LLMs as a
production HTTP service** behind a real load balancer, with SLAs,
multi-tenant LoRA fleets, OpenTelemetry on the latency budget, and
a Hugging Face Hub workflow already in place (HF auth, private
repos, Inference Endpoints parity). Pair with
[`langfuse`](../langfuse/) or [`helicone`](../helicone/) for
end-to-end observability above the model, with
[`litellm`](../litellm/) if you front multiple TGI clusters behind
one router, and with [`bentoml`](../bentoml/) /
[`skypilot`](../skypilot/) for the deployment-orchestration layer.
Skip if your scale is "one developer, one laptop" (use
[`ollama`](../ollama/) / [`llamafile`](../llamafile/) /
[`llama.cpp`](../llama.cpp/)) or if your ask is the *agent loop*
itself, not the model server (TGI is the substrate, not the
substance).
