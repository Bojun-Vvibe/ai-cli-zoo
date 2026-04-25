# lmdeploy

> Snapshot date: 2026-04. Upstream: <https://github.com/InternLM/lmdeploy>

"**A toolkit for compressing, deploying, and serving LLMs.**"
LMDeploy from the InternLM team bundles three things behind one
`lmdeploy` CLI: (1) **TurboMind**, a CUDA inference engine with
persistent-batch scheduling, paged-KV cache, AWQ / GPTQ / SmoothQuant
/ FP8 / INT4-KV quantisation, and tensor parallelism; (2) a
PyTorch-eager backend for models TurboMind doesn't yet have a kernel
for; (3) an OpenAI-compatible HTTP server (`api_server`) plus a
gRPC + Triton ensemble for production. Day-one support lands for
new InternLM / Qwen / Llama / DeepSeek / GLM / MiniCPM releases.

## 1. Install footprint

- `pip install lmdeploy` (Linux + CUDA 11.8 / 12.x wheel; bundles
  TurboMind .so).
- `pip install lmdeploy[serve]` for the FastAPI / gRPC server
  extras; `lmdeploy[all]` for everything.
- Docker: `openmmlab/lmdeploy:latest` (CUDA runtime + Triton).
- One CLI: `lmdeploy chat`, `lmdeploy serve api_server`,
  `lmdeploy serve gradio`, `lmdeploy lite auto_awq`,
  `lmdeploy lite calibrate`, `lmdeploy convert`, `lmdeploy check_env`.
- Python ≥ 3.8, CUDA ≥ 11.8. NVIDIA primary; experimental Ascend
  NPU + AMD ROCm backends.

## 2. Repo + version + license

- Repo: <https://github.com/InternLM/lmdeploy>
- Latest release: **v0.12.3** (2026-04-08)
- License: **Apache-2.0** —
  <https://github.com/InternLM/lmdeploy/blob/main/LICENSE>
- HEAD SHA: `7ee73e746bae97c61aacd3ea8deedfcf05d67070`
- Default branch: `main`
- Language: Python (TurboMind core in C++ / CUDA)

## 3. Models supported

InternLM / InternLM2 / InternLM3, InternVL (vision), Llama 2 / 3 /
3.1 / 3.2 / 3.3, Qwen 1.5 / 2 / 2.5 / 3, Qwen-VL, DeepSeek-V2 / V3 /
R1, Mistral / Mixtral, Gemma 2 / 3, Baichuan 2, Yi, ChatGLM 2 / 3 /
GLM-4, MiniCPM / MiniCPM-V, Phi-3, CodeLlama, plus arbitrary HF
models through the PyTorch-eager backend.

## 4. Simple usage

```bash
# 1. one-shot local chat
lmdeploy chat internlm/internlm3-8b-instruct

# 2. quantise to 4-bit AWQ in one command
lmdeploy lite auto_awq \
  internlm/internlm3-8b-instruct \
  --work-dir ./internlm3-8b-awq

# 3. serve as OpenAI-compatible HTTP on :23333
lmdeploy serve api_server ./internlm3-8b-awq \
  --backend turbomind --tp 1

# 4. point any OpenAI client at it
curl http://localhost:23333/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"internlm3-8b-awq","messages":[{"role":"user","content":"hi"}]}'
```

## 5. Why it's interesting

- **One CLI for the full lifecycle**: `lmdeploy lite` quantises,
  `lmdeploy convert` reshards for tensor parallelism, `lmdeploy
  serve api_server` runs the OpenAI-compatible endpoint — no
  separate quant / convert / serve toolchains.
- **TurboMind's persistent batch + paged KV** lands roughly the
  same throughput numbers as vLLM on Llama-class models while
  also covering the InternLM / InternVL families that are
  first-class here and second-class elsewhere.
- **Quantisation menu out of the box**: AWQ (W4A16), GPTQ,
  SmoothQuant (W8A8), FP8, INT4-KV cache, INT8-KV cache — all
  driven by `lmdeploy lite` subcommands with calibration data
  loaders included, no separate `auto-awq` / `auto-gptq` install.
- **OpenAI-compatible server is the default deploy target**: same
  `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`,
  function-calling, vision-input shape — drop-in for clients that
  already speak OpenAI without writing a translator.

## 6. Caveats

- TurboMind kernels are **CUDA-first**; non-NVIDIA hardware lands
  on the slower PyTorch-eager backend.
- Server defaults bind `0.0.0.0:23333` — firewall before exposing.
- Vision models require a matching `image_processor` shipped with
  the HF repo; bring-your-own-architecture VLMs need a converter.
