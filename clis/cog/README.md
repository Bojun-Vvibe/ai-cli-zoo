# cog

> Snapshot date: 2026-04. Upstream: <https://github.com/replicate/cog>

"**Containers for machine learning.**" `cog` is Replicate's
opinionated container builder for ML models: declare environment
(Python, CUDA, system packages), model entry-point (`predict.py`
with a `Predictor` class), and typed inputs / outputs in a single
`cog.yaml`, then `cog build` produces a runnable Docker image with a
generated FastAPI HTTP server, an OpenAPI schema, an HTTP queue
worker, and the right CUDA / cuDNN userspace libs already wired —
the same image runs locally with `cog predict` and on Replicate's
hosted GPU fleet via `cog push r8.im/owner/model`. The killer move
is that the container is *reproducible* (lockfile-driven Python
deps, deterministic CUDA / Torch matrix selection) so the model card
on Replicate maps 1-to-1 to a Docker image you can `docker pull`
and run on any GPU host.

## 1. Install footprint

- Single Go static binary; install via `brew install cog` (macOS),
  or download from <https://github.com/replicate/cog/releases> for
  Linux / macOS (amd64 + arm64).
- Requires Docker (or Docker-compatible CLI: `podman`, Rancher
  Desktop, Colima, OrbStack) on the host. NVIDIA Container Toolkit
  required on the host for GPU builds and local GPU `cog predict`.
- Sister Python package `cog` (`pip install cog`) is the in-image
  runtime that defines the `Predictor` base class, the typed
  `Input(...)` field, and the FastAPI server contract — installed
  inside the built image, not on the dev host.
- `cog.yaml` is the single source of truth: `build.python_version`,
  `build.python_packages`, `build.system_packages`, `build.cuda`,
  `build.gpu`, `predict: predict.py:Predictor`, `train:
  train.py:Trainer` (optional fine-tune entry-point), `concurrency`,
  `image`.

## 2. Repo + version + license

- Repo: <https://github.com/replicate/cog>
- Latest release: **v0.18.0**
- License: **Apache-2.0** —
  <https://github.com/replicate/cog/blob/main/LICENSE>
- Default branch: `main` (HEAD `aea7995e` at snapshot)
- Language: Go (CLI) + Python (in-image runtime)

## 3. Models supported

Not opinionated about the model — `cog` is the *packaging* layer.
The `Predictor.predict(...)` signature can call any Python ML stack:
`torch`, `transformers`, `diffusers`, `vllm`, `sglang`, `llama-cpp-python`,
`exllamav2`, `whisperx`, `comfyui` workflows, `flax`, `jax`, `tensorflow`,
`onnxruntime`, custom CUDA kernels, etc. The runtime auto-selects a
compatible CUDA / cuDNN / Torch wheel triple from a curated
compatibility matrix — `build.gpu: true` + `build.python_packages:
- torch==2.4.1` resolves the right `+cu124` wheel and base image
without you authoring a Dockerfile.

## 4. MCP support

No. `cog` produces an HTTP / OpenAPI server (FastAPI under the
hood); model invocation is `POST /predictions` with a typed JSON
body matching the `Input` schema. The generated `openapi.json` lets
any OpenAPI-aware client (including LLM tool-use frameworks) call
the prediction endpoint, but there is no first-party MCP wrapper —
front it with [`fastmcp`](../fastmcp/) or any MCP-from-OpenAPI
adapter if you want MCP semantics.

## 5. Sub-agent model

None — `cog` is single-process container packaging, not an agent
runtime. The `train:` entry-point is a separate sibling to
`predict:` for fine-tune jobs (different signature, persists weights
to a configurable output dir), but both are simple Python
callables. Composition (one cog model calling another) is by HTTP
between containers.

## 6. Telemetry stance

- The `cog` CLI itself sends no analytics by default. Build / predict
  flows are local Docker operations.
- `cog push r8.im/owner/model` uploads the image to Replicate's
  private registry — Replicate then sees image layers and any
  invocation metrics from their hosted runtime.
- `cog login` stores a Replicate API token in `~/.config/cog/`.
- The in-image FastAPI server logs to stdout / stderr only; no
  outbound calls beyond what your `Predictor` makes.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Reproducible GPU containers without authoring
a Dockerfile, with the same image running locally and on hosted
infra.** `cog.yaml` resolves a working CUDA + cuDNN + Torch +
Python combination from a curated matrix and bakes it into a
multi-stage image; `cog build -t my-model:latest && docker run
--gpus all -p 5000:5000 my-model:latest` boots the FastAPI
prediction server with `/predictions`, `/health-check`, `/openapi.json`
and a typed input schema generated from the `Predictor.Input`
class, no Dockerfile authored. The same image pushed via `cog push
r8.im/owner/model` becomes a Replicate model with a hosted API,
billing, queueing, and a versioned model page — and a third-party
can `docker pull r8.im/owner/model && cog predict -i prompt=...`
to run *the exact same artefact* on their own GPU box. The
typed-Input contract (`Input(default=..., ge=..., le=...,
choices=[...])`) propagates to OpenAPI docs and Replicate's
auto-generated web UI for free, so the model card *is* the API
contract.

**Weakness.** **Replicate-shaped opinionation.** The compatibility
matrix is curated for what Replicate's GPU fleet actually runs;
exotic CUDA versions, ROCm, or Apple Silicon Metal predictions are
second-class (CPU-only on Mac for local dev, GPU on Linux only).
The `Predictor` lifecycle (`setup()` once, `predict()` per request)
is great for stateless model inference and awkward for streaming /
multi-turn chat (`cog` added streaming + iterators in 0.10+, but
the shape is still request-shaped not session-shaped — for chat UX
prefer [`bentoml`](../bentoml/) / [`litserve`](../litserve/) /
[`gradio`](../gradio/)). `cog.yaml` lockfile semantics are
softer than `uv.lock` / `poetry.lock` (the matrix selection layer
sits above pip's resolver), so reproducibility is "image-level
deterministic" not "wheel-hash deterministic". Push targets are
Replicate-flavoured (`r8.im/...`) but the produced image is a
standard OCI artifact you can also push to GHCR / ECR / GAR / your
own registry and run anywhere `docker run --gpus` works.

**When to choose.** You are **packaging a single model** (an
open-weight LLM, an SDXL / Flux pipeline, a Whisper variant, a
custom diffusion model, a ComfyUI workflow, a fine-tuned vision
model) for an API and you want **one config file → reproducible GPU
container → hosted endpoint** without authoring a Dockerfile or
configuring a serving framework. Default choice for Replicate
deployments; strong choice for any GPU model where the prediction
shape is "one request → one (or streamed) result" and the team
doesn't want to own the Docker / CUDA stack. Pair with
[`bentoml`](../bentoml/) / [`litserve`](../litserve/) when the API
needs more shape than a typed request (auth, batching, multi-model
routing), with [`truss`](../truss/) when the target is Baseten
infra, with [`modal`](../modal/) when you want function-shaped
serverless GPU instead of container-shaped, and with
[`gradio`](../gradio/) / [`comfyui`](../comfyui/) when the demo UI
is the actual deliverable rather than the API.
