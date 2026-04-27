# huggingface_hub

> Snapshot date: 2026-04. Upstream: <https://github.com/huggingface/huggingface_hub>

"**The official Python client for the Hugging Face Hub.**"
`huggingface_hub` is the Python SDK + `hf` CLI that every other entry
in the catalog touches at some point: it resolves model / dataset /
space references like `meta-llama/Llama-3.1-8B-Instruct`, downloads
the right `safetensors` / `gguf` / `onnx` / config files into a
content-addressed cache, and uploads the artefacts your fine-tuner
just produced. It also brokers the `InferenceClient` (serverless +
dedicated endpoints + provider-routed inference across Together,
Replicate, Fal, SambaNova, Cerebras, Nebius, etc.) so a generated
script can call hosted Llama / Qwen / DeepSeek through one Python
surface, and ships the high-throughput `hf_xet` storage backend that
chunks + dedupes large weights on upload + download.

## 1. Install footprint

- `pip install -U huggingface_hub` (core); `pip install
  "huggingface_hub[cli]"` adds the `hf` binary (replaces the legacy
  `huggingface-cli`); `pip install "huggingface_hub[hf_xet]"` adds the
  Rust-backed Xet chunker for fast multi-GB transfers.
- Python ≥ 3.8. Pulls `requests`, `tqdm`, `pyyaml`, `filelock`,
  `packaging`, `fsspec`, `typing_extensions`. No PyTorch / TF / JAX
  required.
- `hf` CLI installed alongside: `hf auth login`, `hf download`,
  `hf upload`, `hf upload-large-folder`, `hf cache scan`,
  `hf cache delete`, `hf jobs run`, `hf repo create`, `hf repo tag`,
  `hf lfs-enable-largefiles`, `hf env`.
- Default cache at `~/.cache/huggingface/hub/` (override via
  `HF_HOME` / `HF_HUB_CACHE`); content-addressed, safe to share
  between processes.

## 2. Repo + version + license

- Repo: <https://github.com/huggingface/huggingface_hub>
- Latest release: **v1.11.0** (PyPI + GitHub)
- License: **Apache-2.0** —
  <https://github.com/huggingface/huggingface_hub/blob/main/LICENSE>
- Default branch: `main` (HEAD `c91e5f50` at snapshot)
- Language: Python

## 3. Models supported

Not a model itself — it is the *transport* for every model on the
Hub. The bundled `InferenceClient` calls:

- Hugging Face hosted Inference (serverless + dedicated Inference
  Endpoints).
- Provider-routed inference (the Hub forwards to Together, Replicate,
  Fal, SambaNova, Cerebras, Nebius, Hyperbolic, Groq, Novita,
  Featherless, Fireworks, Cohere, Black Forest Labs, etc.) — pass
  `provider="together"` to bypass the default routing.
- Any custom Inference Endpoint URL (`InferenceClient(model="https://my-endpoint.aws.endpoints.huggingface.cloud")`).
- Local `text-generation-inference` / `vllm` servers via the
  OpenAI-compatible `/v1/chat/completions` shim
  (`InferenceClient(base_url=..., api_key=...)`).

For *file* operations the surface is universal: any repo on
`huggingface.co` (model, dataset, or Space) and any private
Enterprise Hub.

## 4. MCP support

Yes (client + tiny server). `huggingface_hub.MCPClient` (in
`huggingface_hub.inference._mcp`) connects an `InferenceClient`-driven
chat loop to one or more MCP servers (stdio + HTTP) and surfaces their
tools to the model — the `tiny-agents` example folder uses this to
build sub-100-line agents that talk to filesystem, web-fetch, and
playwright MCP servers. There is no first-party `huggingface_hub
--mcp-server` mode (the Hub itself is an HTTPS API, not an MCP
server), but `tiny-agents` ships a working pattern.

## 5. Sub-agent model

None at the SDK level — `huggingface_hub` is a transport / client.
The bundled `tiny-agents` reference (`pip install
huggingface_hub[mcp]` then `tiny-agents run ./agent.json`) is a
single-loop ReAct agent over `InferenceClient` + MCP, intentionally
small (composition is left to downstream frameworks like
[`smolagents`](../smolagents/), [`agno`](../agno/),
[`langgraph`](../langgraph/), [`crewai`](../crewai/)).

## 6. Telemetry stance

- File operations: anonymous user-agent string
  (`huggingface_hub/<version>; python/<version>; <library>/<version>`)
  is sent on every Hub HTTP request. Disable with
  `HF_HUB_DISABLE_TELEMETRY=1` (also disables `transformers` /
  `datasets` telemetry).
- `InferenceClient` calls go to whichever provider you pointed at;
  default routing through `huggingface.co` means the Hub sees prompts
  + token counts unless you pass `provider="..."` to skip the router
  or call your own endpoint URL directly.
- No analytics SDK, no PostHog, no third-party trackers.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **The universal Hub transport, plus a single
`InferenceClient` that calls every major hosted-LLM vendor through
one routed endpoint.** `hf download owner/repo --include "*.safetensors"`
streams a multi-GB checkpoint into a content-addressed cache that
every other library in the Python AI ecosystem (transformers, vllm,
sglang, lmdeploy, llama.cpp via gguf, candle, mistral.rs, mlx-lm,
sentence-transformers, datasets) reads from for free; `hf upload-large-folder`
turns a 200 GB fine-tune output dir into a resumable, deduped Xet
upload; and `InferenceClient(model="meta-llama/Llama-3.1-70B-Instruct",
provider="auto").chat_completion(...)` lets a script try Hugging
Face's hosted inference first and transparently fall back to a
provider partner without rewriting the call site. The `hf jobs run`
subcommand (new in the 1.x line) submits a Docker-shaped job to
hosted GPU infra from the same CLI that downloaded the model.

**Weakness.** **Account-tier coupling for serverless inference.** The
default `provider="auto"` routing is rate-limited and quota-gated;
heavy production use needs a dedicated Inference Endpoint or a
direct provider key, which puts you back in the per-vendor SDK story
the unified client was meant to abstract. The legacy
`huggingface-cli` ↔ `hf` rename means stale tutorials and
`Makefile`s still call the old binary (kept as a deprecated alias).
Some Xet features are still Hub-side gated (org needs Xet enabled).
The library leaks `requests` exceptions through its API surface,
which complicates async-first codebases (the async path
`AsyncInferenceClient` exists but is a separate surface).

**When to choose.** Always — if you touch the Hugging Face Hub from
Python, this *is* the client. Choose it explicitly when you want
**one CLI for download + upload + cache management + serverless
inference + Spaces + jobs**, when you want the `InferenceClient`'s
provider-routing as your hedge against single-vendor outages, or when
you need the Xet chunker for large-checkpoint workflows. Skip *only*
the high-level inference surface if you have a fixed provider already
and prefer that vendor's SDK directly — but you'll still want
`huggingface_hub` underneath for weight transport.
