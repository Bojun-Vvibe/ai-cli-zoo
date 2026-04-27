# candle

> Snapshot date: 2026-04. Upstream: <https://github.com/huggingface/candle>

A minimalist ML framework written in Rust, by Hugging Face — the
**"PyTorch in Rust"** that actually runs production inference for
the kinds of models the catalog cares about (LLaMA / Mistral / Qwen
/ Phi / Gemma / Whisper / Stable Diffusion / SDXL / Flux). Same
shape as PyTorch — `Tensor`, `nn::Module`, `VarBuilder`, `safetensors`
loading — but it compiles to a single static binary you can drop on
a server, an embedded device, or a WASM bundle with no Python
runtime.

It is the catalog's reference for **"production inference, no
Python"** — useful when the deployment target is a Tauri desktop
app, a server-side function with strict cold-start budgets, an
edge device, or a WebAssembly bundle.

## 1. Install footprint

- `cargo add candle-core candle-nn candle-transformers` — pulls a
  ~30 MB compile-time-only dependency tree; the resulting binary is
  in the single-MB range.
- Backends: CPU (Rayon-parallel SIMD), CUDA (NVIDIA),
  Metal (Apple Silicon), WebGPU (via `candle-wgpu`, third-party).
  Enable via Cargo features: `candle-core = { version = "0.10",
  features = ["cuda"] }` or `["metal"]`.
- Optional crates: `candle-flash-attn` (FlashAttention 2 kernels for
  CUDA), `candle-onnx` (ONNX runtime), `candle-datasets` (HF
  datasets loader), `candle-pyo3` (Python bindings).
- WASM target works (`wasm32-unknown-unknown`) — the
  [candle-wasm-examples](https://github.com/huggingface/candle/tree/main/candle-wasm-examples)
  ship Whisper / LLaMA / SAM / YOLO inference running entirely in a
  browser tab with the model weights downloaded once and cached in
  IndexedDB.

## 2. Repo, version, license

- Repo: <https://github.com/huggingface/candle>
- Version checked: **0.10.1** (latest tag as of 2026-04; upstream
  releases are tag-only — no GitHub Releases page).
- HEAD pinned at this snapshot:
  `5447a8779871654e542e65693e7d26f1e33b215f`.
- License: **Apache-2.0 OR MIT** dual-licensed (consumer's choice).
  License files at
  [`LICENSE-APACHE`](https://github.com/huggingface/candle/blob/main/LICENSE-APACHE)
  (blob sha `261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64`) and
  [`LICENSE-MIT`](https://github.com/huggingface/candle/blob/main/LICENSE-MIT)
  (blob sha `31aa79387f27e730e33d871925e152e35e428031`) — the standard
  Rust ecosystem dual-license pattern.

## 3. What it actually does

Candle is layered:

1. **`candle-core`.** `Tensor`, `Device` (CPU / Cuda / Metal),
   `DType` (F32 / F16 / BF16 / I64 / U8 / U32), automatic
   broadcasting, in-place ops, `safetensors` load / save, lazy
   evaluation for kernel fusion. ~15k lines of Rust.
2. **`candle-nn`.** Linear / Conv / LayerNorm / RMSNorm / RoPE /
   GroupNorm / dropout / activation primitives. `VarBuilder` for
   loading `safetensors` checkpoints into `nn::Module` trees by
   path-prefix convention. The shape PyTorch users expect.
3. **`candle-transformers`.** Reference implementations of every
   model the HF Hub cares about — LLaMA-3.x, Mistral / Mixtral, Qwen
   2.5 / 3, Phi-3 / 4, Gemma 2 / 3, DeepSeek-V3, Falcon, MPT,
   StarCoder, BLOOM, Whisper, T5, BERT, CLIP, Stable Diffusion 1.5
   / 2.x / XL / 3, Flux.1 (Schnell / Dev), SAM, YOLO-v8, DINOv2,
   ConvNeXt — usually within weeks of release on the HF Hub.
4. **`candle-examples`.** Runnable binaries — `cargo run --release
   --bin llama -- --prompt "Hello"` downloads the LLaMA-3 weights
   from HF Hub, loads them with `VarBuilder`, runs decode on the
   chosen backend.
5. **Quantisation.** `candle-core::quantized` reads GGUF and GGML
   files directly; the Q4_0 / Q4_1 / Q5_0 / Q5_1 / Q8_0 / Q4_K /
   Q5_K / Q6_K / Q8_K formats from `llama.cpp` are decoded natively,
   so a `tinyllama-1.1b.Q4_K_M.gguf` runs without going through
   `llama.cpp` first.

## 4. MCP support

None first-party — it's an inference framework. The integration
boundary is "expose an OpenAI-compatible HTTP endpoint", which
[mistral.rs](https://github.com/EricLBuehler/mistral.rs) (built on
candle) does at the binary level; an MCP-aware client points at
that endpoint.

## 5. Sub-agent model

N/A. Multi-device parallelism is `Tensor::to_device` placement
plus user-managed sharding; tensor-parallel inference for very
large models is in candle-transformers for a few specific shapes
(LLaMA, Mixtral) but is not the framework's first-class story —
that lives in vLLM / SGLang / TensorRT-LLM upstream.

## 6. Telemetry stance

Off. No analytics in the framework or examples. Egress is the HF
Hub model fetch on first run (cacheable to a mounted directory for
air-gapped deployments) and whatever the GPU vendor driver does on
its own.

## 7. Token / context strategy

The bundled examples use a hand-rolled KV cache (one `Tensor` per
layer per role, concatenated along the sequence dim per decode
step). It is correct and clear but not paged — for serious
high-throughput serving, the relevant production project is
**mistral.rs** (downstream of candle), which adds paged-attention,
continuous batching, dynamic LoRA, prefix caching, and an
OpenAI-compatible HTTP server on top of candle's tensor + nn layer.

## 8. Hot keybinds

None — library + example binaries. The CLI flags in
`candle-examples` follow the HF convention: `--model-id <hub_id>`,
`--prompt`, `--temperature`, `--top-p`, `--sample-len`,
`--repeat-penalty`, `--cpu` to force the CPU backend.

## 9. Killer feature, weakness, when to choose

**Killer feature.** A real production-grade inference framework
written in Rust, with **no Python runtime** in the deployed binary,
that ships reference implementations of every architecture the HF
Hub cares about within weeks of release — the only entry in the
catalog where "compile to a 5 MB static binary that runs LLaMA-3-8B
on CUDA" is a `cargo build --release` away. Native GGUF support
means the same framework runs full-precision `safetensors` and
quantised GGUF without a separate runtime. WASM target works for
Whisper / LLaMA / SAM in-browser without a server.

**Weakness.** Pre-1.0 — the API still moves between minor versions
(0.9 → 0.10 broke a few `VarBuilder` paths). For a single-model
serving binary that's fine; for a multi-model long-lived deployment
pin a commit. Performance at high batch sizes / long contexts
trails vLLM + PagedAttention by a meaningful factor — candle is the
right answer for low-latency low-batch (1–4 concurrent requests)
serving on edge / desktop / Apple Silicon, the wrong answer for
1000-concurrent-user serving on H100s where the production path is
[`vllm`](../vllm/) / [`sglang`](../sglang/) / [`text-generation-inference`](../text-generation-inference/).
The training story exists (`candle-nn` has optimisers and autograd)
but is not the framework's focus — almost nobody trains on candle.

**When to choose.**
- The deployment target is a **Tauri desktop app** / **CLI binary**
  / **Lambda function** / **edge device** / **browser via WASM** —
  Python is a non-starter, and you need a real ML framework not a
  thin ONNX-runtime wrapper.
- You're on **Apple Silicon** and want first-class Metal acceleration
  in a Rust binary — candle's Metal backend is solid.
- You need to run **GGUF quantised models** without bringing in
  `llama.cpp` as a separate runtime.
- The team is already shipping Rust services and a Python ML server
  is the architectural odd-one-out.

**When to skip.**
- You're training models — use PyTorch.
- You're serving 100+ concurrent users on NVIDIA H100 — use [`vllm`](../vllm/).
- You want the broadest pretrained-model ecosystem with
  `from_pretrained` ergonomics — use `transformers` (Python).
- You need batched serving with paged-attention and dynamic LoRA out
  of the box — use [`mistral.rs`](https://github.com/EricLBuehler/mistral.rs)
  (built on candle, but the production server, not the framework).

## 10. Compared to neighbors in the catalog

| Tool | Lang | Primary job | Best for | Training? |
|------|------|-------------|----------|-----------|
| candle | Rust | DL framework + inference | Edge / desktop / WASM / Rust services | Possible, not focus |
| [tinygrad](../tinygrad/) | Python | Hackable DL framework | Reading the source / AMD / Metal | Yes |
| [mlx-lm](../mlx-lm/) | Python | Apple Silicon LLM serving | Apple-only inference + LoRA | Yes (LoRA) |
| [llama.cpp](../llama.cpp/) | C++ | LLM inference | CPU + every quant + every backend | No |
| [vllm](../vllm/) | Python | High-throughput LLM serving | NVIDIA, 100+ concurrent users | No |

Decision shortcut:
- "Single static binary, no Python, runs LLaMA-3 on CUDA / Metal /
  CPU / WASM" → **candle**.
- "Read a DL framework end-to-end" → [`tinygrad`](../tinygrad/).
- "Apple Silicon, Python is fine" → [`mlx-lm`](../mlx-lm/).
- "Production serving on H100s with 100+ concurrent users" →
  [`vllm`](../vllm/).
