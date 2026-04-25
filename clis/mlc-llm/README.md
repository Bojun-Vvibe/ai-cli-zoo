# mlc-llm

> Snapshot date: 2026-04. Upstream: <https://github.com/mlc-ai/mlc-llm>

"**Universal LLM Deployment Engine with ML Compilation.**"
MLC-LLM (Machine Learning Compilation for LLMs) is the
deployment-time peer of [`llama.cpp`](../llama.cpp/) for projects
that want to **compile** a model graph to native code per target
device rather than interpret it through a runtime kernel library.
The pipeline is: HuggingFace weights → `mlc_llm convert_weight`
(quantise + repack) → `mlc_llm gen_config` (chat template + KV
shape) → `mlc_llm compile` (TVM Unity / Relax IR → device-
specific shared library) → `mlc_llm chat` / `mlc_llm serve` /
embed in iOS / Android / WebGPU app. The output is a tiny self-
contained library you ship — no Python at runtime, no CUDA
runtime if you targeted Metal / Vulkan / WebGPU.

## 1. Install footprint

- `pip install --pre mlc-llm-nightly mlc-ai-nightly` (CPU/CUDA
  pre-built wheels for Linux / macOS / Windows). Per-backend
  wheels exist for CUDA 11.8 / 12.x, ROCm, Vulkan, Metal.
- CLI: `mlc_llm` with subcommands `convert_weight`, `gen_config`,
  `compile`, `chat`, `serve`, `bench`, `package`, `router`.
- Build-from-source: `cmake` + LLVM + TVM Unity submodule; ~30-min
  build on a workstation. Required if you target an exotic device
  not covered by the wheels.
- Mobile / Web outputs are produced by the `package` subcommand and
  shipped via the companion projects `mlc-llm/ios`, `mlc-llm/android`,
  `web-llm` (WebGPU in-browser).
- Python ≥ 3.10. The runtime libraries the compiler emits have
  zero Python dependency.

## 2. Repo + version + license

- Repo: <https://github.com/mlc-ai/mlc-llm>
- Latest tag: **v0.20.dev0** (rolling dev tag; project ships
  nightly wheels rather than tagged releases)
- License: **Apache-2.0** —
  <https://github.com/mlc-ai/mlc-llm/blob/main/LICENSE>
- HEAD SHA: `d1ea69a87280e821611643958bfec385b62dafd3`
- Default branch: `main`
- Language: Python (compiler + driver), C++ (runtime), with
  TVM-generated kernels per device

## 3. Models supported

Llama 1 / 2 / 3 / 3.1 / 3.2, Mistral / Mixtral, Qwen 1.5 / 2 /
2.5, Phi 1.5 / 2 / 3, Gemma 1 / 2, DeepSeek / DeepSeek-Coder,
Yi, Baichuan, ChatGLM 2 / 3, InternLM 1 / 2, RWKV 5 / 6,
StableLM, MiniCPM, Bunny / LLaVA / Phi-3-Vision (multimodal).
Quantisation: q3f16_1, q4f16_1, q4f32_1, q0f16, q0f32 (the
`qXfY_Z` codes encode weight bit-width / activation precision /
group-size variant). Custom architectures are added by writing a
Python model definition in TVM Relax — non-trivial but
documented.

## 4. Simple usage

```bash
# Convert HF weights -> MLC weights (4-bit group-quant)
mlc_llm convert_weight ./hf-models/Llama-3.2-3B-Instruct \
  --quantization q4f16_1 -o ./dist/llama-3.2-3b-q4f16_1

# Generate per-device chat config + KV cache plan
mlc_llm gen_config ./hf-models/Llama-3.2-3B-Instruct \
  --quantization q4f16_1 --conv-template llama-3 \
  -o ./dist/llama-3.2-3b-q4f16_1

# Compile to a target-specific shared library
mlc_llm compile ./dist/llama-3.2-3b-q4f16_1/mlc-chat-config.json \
  --device metal \
  -o ./dist/llama-3.2-3b-q4f16_1/llama-3.2-3b-q4f16_1-metal.so

# Chat in the terminal
mlc_llm chat ./dist/llama-3.2-3b-q4f16_1

# OpenAI-compatible HTTP server
mlc_llm serve ./dist/llama-3.2-3b-q4f16_1 --host 0.0.0.0 --port 8000
```

## 5. Why it's interesting

- **Compile, don't interpret.** A graph compiler (TVM Unity /
  Relax) lowers the model to a device-specific shared library
  with kernels picked / fused for the actual hardware — Metal
  intrinsics on Apple Silicon, CUDA Tensor Cores on NVIDIA,
  ROCm wavefront tuning on AMD, SPIR-V on Vulkan, WebGPU shaders
  in the browser. The same toolchain emits an iOS / Android /
  WebAssembly artefact from the same source recipe.
- **WebGPU in-browser inference is unique.** `web-llm`
  (sister project) runs Llama-class models entirely client-side
  in Chrome / Edge / Safari Tech Preview without a server round
  trip — no other entry in this catalog hits that target.
- **`mlc_llm serve`** exposes a vLLM-style OpenAI-compatible
  endpoint with continuous batching, paged-KV cache, and prefix
  caching, so a compiled model can stand in for vLLM on
  hardware vLLM doesn't cover well (Metal, Vulkan, ROCm of older
  vintage, niche embedded GPUs).
- **`mlc_llm package`** bundles weights + library + tokenizer
  into a directory layout that the iOS / Android / web SDKs
  consume directly — the "ship an LLM in your app store binary"
  path most projects don't have.

## 6. Caveats

- The conversion pipeline is **three steps** (`convert_weight` →
  `gen_config` → `compile`); there is no `mlc_llm pull <hf-id>`
  one-liner equivalent to `ollama pull`. Expect to read the docs.
- Compile time is non-trivial (minutes per (model, quant, device)
  combination); cache the artefacts.
- Model coverage trails [`llama.cpp`](../llama.cpp/) and
  [`vllm`](../vllm/): if a brand-new architecture dropped on
  HuggingFace yesterday, MLC-LLM is rarely the same-week first
  mover.
- Adding a brand-new architecture means writing TVM Relax — that
  is closer to "porting a kernel" than "PR-ing a model class",
  and the contributor surface is smaller.
- Project names overlap with the broader MLC ecosystem
  (`mlc-llm`, `mlc-ai`, `web-llm`, `mlc-chat`); read the README
  to map subprojects.

## 7. When to choose

You want one source recipe to deploy the **same model across
desktop, mobile, and web**, you care about Metal / Vulkan /
WebGPU as first-class targets, and you are willing to compile
artefacts ahead of time in exchange for a tiny self-contained
runtime. Particularly compelling for **on-device inference in
shipped apps** (iOS, Android, browser extensions) where the
runtime size and lack of Python at runtime matter. Pair with
[`llama.cpp`](../llama.cpp/) when you'd rather have a
runtime-interpreted GGUF file than a compiled `.so`. Pair with
[`vllm`](../vllm/) when your deployment target is a Linux box
with NVIDIA GPUs and you want maximum tokens/sec without
dealing with TVM at all. Pair with
[`mlx-lm`](../mlx-lm/) on the Apple side if you'd prefer the
Apple-native MLX framework over a TVM-compiled Metal kernel.
