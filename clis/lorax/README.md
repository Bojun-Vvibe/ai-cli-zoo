# lorax

> Snapshot date: 2026-04. Upstream: <https://github.com/predibase/lorax>

"**Multi-LoRA inference server that scales to 1000s of fine-tuned
LLMs.**" Lorax is a single GPU inference server that loads one
shared base model (Llama / Mistral / Qwen / Gemma / Phi families) and
swaps thousands of LoRA adapters in and out of the same forward pass
on demand, so a fleet of per-customer / per-task fine-tunes serves
from a single weight-of-base footprint instead of N × full-model
deployments. Adapters are pulled at request time from a Predibase
endpoint, the Hugging Face Hub, S3, GCS, Azure blob, or local disk
by passing `adapter_id` in the request body, then cached on the
server with an LRU policy so the second request to the same adapter
is hot. Heterogeneous continuous batching means request A
(adapter `customer-42`) and request B (adapter `customer-9001`) and
request C (no adapter, base model) can all share one batch step
without per-adapter queueing. Exposes both a native HTTP API and an
OpenAI-compatible `/v1/chat/completions` endpoint, plus a Python /
JS client.

## 1. Install footprint

- Container-first: `ghcr.io/predibase/lorax:main` Docker image
  (CUDA 12.x base, ~12 GB), single command boots the server with
  `--model-id <hf-repo>` plus an optional `--source pbase` /
  `s3` / `gcs` / `hub` / `local` adapter source.
- Python client: `pip install lorax-client` for a typed wrapper
  around the HTTP API; standard `openai` / `httpx` clients also
  work against the OpenAI-compatible `/v1/...` surface.
- NVIDIA GPU required (Ampere or newer in practice — A10G / A100 /
  H100 / L4 / L40S; consumer 30/40-series work for smaller bases).
  No CPU / Apple Silicon / AMD path in the supported matrix.
- Built on top of `text-generation-inference` internals (FlashAttn2,
  Paged-KV, continuous batching) plus a custom Punica-style
  multi-adapter kernel (`Segmented Gather Matrix-Vector
  Multiplication`, SGMV) that batches per-token adapter applies in
  one CUDA kernel.

## 2. Repo + version + license

- Repo: <https://github.com/predibase/lorax>
- Latest release: **lorax-0.4.0** (Helm chart tag; server version
  follows the rolling `:main` Docker tag, no separate semver release
  cadence on the binary)
- HEAD SHA: `b5a9e38dc9479ca664bbaac9ae949e27e3c30832`
- License: **Apache-2.0** —
  <https://github.com/predibase/lorax/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (orchestrator) + Rust (router) + custom CUDA /
  Triton kernels

## 3. Models supported

Any base model with FlashAttn2 + Paged-KV implementations in the
HF `text-generation-inference` lineage that Lorax forks: Llama 2 /
3 / 3.1 / 3.2 / 3.3, Mistral / Mixtral, Qwen 1.5 / 2 / 2.5,
Gemma 1 / 2, Phi-2 / Phi-3, Falcon, MPT, GPT-NeoX, BLOOM, plus the
embedding models in the same family for retriever fine-tunes.
Adapters: any LoRA produced by `peft` against one of the above
bases — full-rank LoRA, QLoRA-trained adapters, and DoRA all load.
Tokenizer must match the base; attempting to load an adapter
trained against a different base returns a structured 4xx instead
of silently corrupting output. No support for full-weight fine-tunes
(by design — that's what `vllm` / `tgi` are for); no support for
adapters that mutate non-LoRA params (embedding-resize, added
tokens) — the README documents the exact `peft` config shape
that loads cleanly.

## 4. MCP support

No. Lorax is an inference *runtime*, not an agent or tool host —
it exposes an OpenAI-compatible HTTP surface that any MCP-aware
agent runtime ([`opencode`](../opencode/), [`claude-code`](../claude-code/),
[`crush`](../crush/), [`continue`](../continue/),
[`fast-agent`](../fast-agent/), [`pydantic-ai`](../pydantic-ai/))
can point at as the model backend. MCP composition happens one
level up.

## 5. Sub-agent model

None — runtime layer. Concurrency is at the request level: the
continuous-batching scheduler interleaves tokens from N concurrent
requests in one forward pass, with per-request `adapter_id`
selection happening inside the SGMV kernel. There is no agent loop
inside Lorax.

## 6. Telemetry stance

Off. The server runs locally inside your container; egress is
exactly the model + adapter pulls you configure (HF Hub on first
load of the base, plus your chosen adapter source on each unseen
adapter request) and inbound HTTP from your clients. No analytics
service, no phone-home. Optional Prometheus `/metrics` endpoint for
your own monitoring.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **One GPU, thousands of adapters, single
batch.** Loading 1000 LoRA adapters against a Llama-3.1-8B base
adds ~4 GB to the base's ~16 GB FP16 footprint (rank-16 adapters,
~4 MB each) instead of the 16 TB you'd spend deploying 1000 full
8B fine-tunes — and the SGMV kernel applies the right adapter
per request *in the same batch step*, so request `customer-A`
and request `customer-B` don't serialize. The
`adapter_id` field in the OpenAI-compatible request body is the
entire UX — no per-tenant deployment to provision, no warmup, no
spin-down. For SaaS shapes where every customer has their own fine-
tune, this collapses serving cost from O(tenants) to O(1).

**Weakness.** **NVIDIA-only and base-model-locked.** No AMD / Apple
/ CPU path; no support for full-weight fine-tunes (those need
[`vllm`](../vllm/) / [`sglang`](../sglang/) /
[`lmdeploy`](../lmdeploy/) per fine-tune); no support for adapters
trained against a different base than the one loaded at boot
(switching the base is a server restart, not a hot swap). The
multi-adapter throughput advantage shrinks as the *number of
distinct adapters per batch* grows — at extreme adapter heterogeneity
(every request a different adapter) you trend toward single-stream
performance per request. Adapter cold-start (first pull from
remote source) adds latency; pre-warm via `POST /load_adapter`
if your tail-latency budget is strict. Project velocity has
slowed — the rolling `:main` tag is the canonical install rather
than a tagged server release, so pin a specific image SHA in
production.

**When to choose.** You have a base-model-fixed deployment with N
fine-tuned variants (per-customer, per-task, per-domain) and want
one GPU instead of N GPUs. Pair upstream with a fine-tune pipeline
that emits `peft`-shaped LoRA checkpoints to an S3 / HF / Predibase
source, and downstream with any OpenAI-compatible client (or an
agent runtime). Skip if you only have one or two model variants
(use [`vllm`](../vllm/) /
[`sglang`](../sglang/) / [`openllm`](../openllm/) with one
deployment per variant), if you need full-weight fine-tunes (Lorax
can't serve those), or if your hardware is not NVIDIA. Pair with
[`unsloth`](../unsloth/) for the fine-tune side — the LoRA adapters
unsloth produces drop into Lorax with no conversion.
