# llama.cpp

> Snapshot date: 2026-04. Upstream: <https://github.com/ggml-org/llama.cpp>

"**LLM inference in C/C++.**" `llama.cpp` is the reference
portable inference engine for open-weights LLMs: a single
self-contained C/C++ codebase (built on the `ggml` tensor
library) that runs Llama / Qwen / Mistral / Mixtral / Phi /
Gemma / DeepSeek / Granite / Falcon / and ~100 other architectures
on any CPU, plus optional GPU back-ends for CUDA, Metal, Vulkan,
ROCm, SYCL, Kompute, MUSA, and CANN. The **GGUF** file format
(invented here) and the **K/I-quants** (2 / 3 / 4 / 5 / 6 / 8-bit
weight quantisation tuned for CPU-friendly memory layout) are now
the de-facto interchange format for self-hosted LLMs — when
[`ollama`](../ollama/), [`llamafile`](../llamafile/),
[`localai`](../localai/), [`koboldcpp`], or [`lm-studio`] say
"GGUF model", they mean the artefact this project produces, and
they almost certainly link `libllama` under the hood.

## 1. Install footprint

- Binaries: `llama-cli`, `llama-server`, `llama-quantize`,
  `llama-bench`, `llama-perplexity`, `llama-tokenize`,
  `llama-embedding`, `llama-gguf-split`, `llama-imatrix`,
  `llama-llava-cli`, `llama-mtmd-cli` (multimodal), and friends —
  all in the same build tree.
- Build: `cmake -B build && cmake --build build -j` (CPU-only).
  GPU back-ends are CMake flags: `-DGGML_CUDA=ON`, `-DGGML_METAL=ON`
  (default on macOS), `-DGGML_VULKAN=ON`, `-DGGML_HIPBLAS=ON`
  (ROCm), `-DGGML_SYCL=ON`. Pre-built releases ship binaries for
  Linux / macOS / Windows on x86_64 and ARM64.
- Brew: `brew install llama.cpp`. Nix: `nixpkgs.llama-cpp`.
  Docker: `ghcr.io/ggml-org/llama.cpp:full`. Python bindings live
  out-of-tree in `llama-cpp-python`.
- Zero runtime deps — the binary is self-contained. Models are
  GGUF files you `wget` from HuggingFace.

## 2. Repo + version + license

- Repo: <https://github.com/ggml-org/llama.cpp>
- Latest release: **b8933** (rolling daily release tag, 2026-04)
- License: **MIT** —
  <https://github.com/ggml-org/llama.cpp/blob/master/LICENSE>
- HEAD SHA: `dcad77cc3b0865153f486327064fb0320a57a476`
- Default branch: `master`
- Language: C++ (with C and CUDA / Metal / Vulkan kernels)

## 3. Models supported

The supported-architecture list is the longest in the catalog:
LLaMA 1 / 2 / 3 / 3.1 / 3.2 / 3.3 / 4, Mistral / Mixtral,
Qwen 1 / 1.5 / 2 / 2.5 / 3 (incl. MoE, Coder, VL, Audio),
Phi 1.5 / 2 / 3 / 3.5 / 4, Gemma 1 / 2 / 3, DeepSeek-V2 / V3 /
R1 / Coder, Granite / Granite-MoE, Falcon, Bloom, Baichuan,
ChatGLM, InternLM, Yi, Command-R / R+, StarCoder 1 / 2,
RefactGPT, OLMo, OLMoE, BERT (encoder-only, for embeddings),
Jamba (state-space hybrid), RWKV 4 / 5 / 6, Mamba / Mamba2,
T5 / FLAN-T5, plus multimodal (LLaVA, MiniCPM-V, MobileVLM,
Bunny, Yi-VL, Qwen2-VL, Qwen2.5-VL, Pixtral, Gemma-3-Vision,
SmolVLM, GLM-4V) and audio (Qwen2-Audio, Whisper via
`whisper.cpp` sister project). Quantisation: Q2_K through Q8_0,
IQ1_S through IQ4_XS (importance-matrix quants), plus FP16 / BF16
/ FP32. Almost any GGUF you find on HuggingFace works out of the
box.

## 4. Simple usage

```bash
# 1. Grab a GGUF model
wget https://huggingface.co/.../Qwen2.5-7B-Instruct-Q4_K_M.gguf

# 2. One-shot completion
llama-cli -m Qwen2.5-7B-Instruct-Q4_K_M.gguf \
  -p "Explain PagedAttention in two sentences." -n 200

# 3. OpenAI-compatible HTTP server
llama-server -m Qwen2.5-7B-Instruct-Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 -c 8192 -ngl 99

# 4. Quantise an FP16 model down to Q4_K_M
llama-quantize ./model-f16.gguf ./model-Q4_K_M.gguf Q4_K_M

# 5. Convert HF safetensors -> GGUF
python convert_hf_to_gguf.py /path/to/hf-model --outfile model.gguf
```

## 5. Why it's interesting

- **Universal hardware reach.** The same binary serves a Llama-3
  on a Raspberry Pi, an M-series MacBook (Metal), an NVIDIA
  workstation (CUDA), an AMD box (ROCm / Vulkan), an Intel Arc
  laptop (SYCL / Vulkan), and a phone (Android NDK / iOS). No
  other engine in this catalog spans that envelope.
- **GGUF + K/I-quants.** A 70 B model that needs 140 GB at FP16
  fits in 40 GB at Q4_K_M with single-digit perplexity loss, and
  the file is one self-describing blob you can `scp` to any box.
  This is why the "self-hosted LLM" ecosystem standardised on it.
- **`llama-server` is OpenAI-compatible** out of the box: serves
  `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`,
  `/v1/rerank`, plus its own `/completion` and `/slots` endpoints
  for slot-level KV-cache control.
- **Speculative decoding, parallel slots, prompt caching, grammar-
  constrained sampling (GBNF), JSON-schema sampling, LoRA hot-swap**
  — all built in, all configurable via CLI flags.

## 6. Caveats

- Pure-throughput on a fat NVIDIA box: [`vllm`](../vllm/) and
  [`sglang`] generally win on tokens/sec for the same model on
  the same H100, because their batch schedulers are tuned for
  GPU-memory-bound serving in a way `llama.cpp` is not.
- Build flags matter: a CPU-only build is great until you wonder
  why the GPU is idle — you have to opt in to the right backend
  at compile time (`GGML_CUDA`, `GGML_METAL`, …).
- Multi-GPU tensor parallelism exists (`--split-mode row` /
  `--main-gpu`) but is less mature than `vllm`'s; pipeline
  parallelism is single-machine only.
- The CLI surface is wide and historically inconsistent — flags
  have been renamed across releases; pin a specific release tag
  in scripts.

## 7. When to choose

You want **one inference binary that runs on every machine you
own**, you care about CPU and Apple-Silicon paths as much as
NVIDIA, and you want first-class GGUF + quantisation tooling.
Pair with [`ollama`](../ollama/) or [`llamafile`](../llamafile/)
if you'd rather not deal with raw flags — both are thin wrappers
over `libllama`. Pair with [`outlines`](../outlines/) or
[`guidance`](../guidance/) for structured-output decoding (or use
the built-in GBNF grammar). Skip in favour of [`vllm`](../vllm/)
or [`lmdeploy`](../lmdeploy/) if you have a serious NVIDIA fleet
and your only goal is maximum tokens/sec on one architecture;
skip in favour of [`mlx-lm`](../mlx-lm/) if you are Apple-Silicon-
only and want the native MLX path; skip in favour of
[`openllm`](../openllm/) if you want a YAML-pinned bento
abstraction over the engine.
