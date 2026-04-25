# bentoml

> Snapshot date: 2026-04. Upstream: <https://github.com/bentoml/BentoML>

"**Turn any Python ML / LLM function into a production HTTP +
gRPC service, packaged as a reproducible OCI image, deployable to
Kubernetes or BentoCloud with one command.**" `bentoml` is the
canonical "model → service" framework: you decorate a Python class
with `@bentoml.service`, declare inputs / outputs as Pydantic
schemas, and the framework generates a FastAPI / Starlette server,
auto-batching, adaptive micro-batching, request-level GPU
scheduling, OpenAPI docs, Prometheus metrics, OpenTelemetry
tracing, multi-replica runner topology, and a Dockerfile / OCI
image. The same `service.py` runs locally for dev (`bentoml serve`),
builds into a versioned `bento` artefact (`bentoml build`),
containerises (`bentoml containerize`), and deploys to BentoCloud
or any Kubernetes cluster (`bentoml deploy`) — no rewrite per
target. It is the substrate underneath
[`openllm`](../openllm/), [`litserve`](../litserve/) is the
"smaller, single-file" alternative, and the project also publishes
the OpenLLM CLI for the LLM-specific subset of this surface.

## 1. Install footprint

- `pip install bentoml` (Python ≥ 3.9). Pulls FastAPI, Starlette,
  Uvicorn, Pydantic v2, click, rich, prometheus-client,
  opentelemetry-api, attrs, deepmerge, simple-di, and the
  bentoml runner protocol. ~80 MB resolved.
- Single binary on PATH: `bentoml`. Subcommand surface:
  `serve`, `build`, `containerize`, `deploy`, `list`, `delete`,
  `models`, `cloud`, `start-runner-server`.
- Optional extras: `pip install "bentoml[grpc]"` for the gRPC
  server, `pip install "bentoml[aws]"` / `[gcp]` / `[azure]` for
  cloud-storage model-store backends, `pip install "bentoml[io]"`
  for the extended IO descriptors (image / audio / video /
  pandas-DataFrame / numpy-NDArray).
- Container build is pure-Python (no Docker daemon required for
  `bentoml build`; only `bentoml containerize` shells out to
  `docker` / `podman` / `buildah`).

## 2. Repo + version + license

- Repo: <https://github.com/bentoml/BentoML>
- Latest release: **v1.4.38** (2026-04-02)
- License: **Apache-2.0** —
  <https://github.com/bentoml/BentoML/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (with a small Rust acceleration module for the
  request router on the `1.4` line)

## 3. What it serves

- **Any Python function**: classical sklearn / xgboost / lightgbm /
  catboost models, PyTorch / TensorFlow / JAX inference, Hugging
  Face Transformers / Diffusers pipelines, vLLM / TensorRT-LLM /
  SGLang LLM backends, ONNX Runtime sessions, custom business-
  logic Python.
- **Multi-model pipelines**: declare multiple `@bentoml.service`
  classes in one bento and wire them with `bentoml.depends()` —
  BentoML places each on its own pod with its own GPU/CPU shape
  and auto-scales independently.
- **Streaming + async**: `async def` endpoints + Server-Sent
  Events / WebSocket / chunked-transfer for token streaming;
  built-in OpenAI-compatible chat-completion shim for LLM
  services.
- **Adaptive batching**: `@bentoml.api(batchable=True)` enables
  request-level dynamic batching tuned by latency target —
  critical for GPU-bound LLM / vision workloads.
- **Model store**: `bentoml.models.create()` versions weights
  with content-addressable hashing, optional S3 / GCS / Azure
  Blob backend, signed-URL pull at container start.

## 4. When to use it

- You want a **production-shaped HTTP / gRPC service for an ML
  model** with batching, metrics, tracing, OpenAPI, and a
  reproducible OCI image — without writing FastAPI + Dockerfile +
  Helm chart by hand.
- You need **multi-model topology**: a vision model + an LLM + a
  business-logic pre/post-processor as separate replicas with
  separate GPU shapes, wired together by typed Python calls.
- You want the **same artefact (the bento) to run locally,
  on-prem K8s, and on BentoCloud** — the build and deploy CLIs
  pin Python version, OS base, CUDA version, and dependency hashes
  so dev / staging / prod do not drift.
- You need **adaptive micro-batching** to hit GPU utilisation
  targets on bursty traffic.
- You want **OpenLLM under the hood**: `openllm serve llama3.3:70b`
  is a thin BentoML service; if you need to extend / customise it,
  drop down to BentoML directly.

## 5. When NOT to use it

- You want **the smallest possible single-file LLM server** —
  reach for [`litserve`](../litserve/). BentoML adds packaging /
  build / cloud-deploy machinery you may not need for a one-box
  demo.
- You only serve **one open-source LLM as an OpenAI-compatible
  endpoint** with no custom business logic — [`openllm`](../openllm/)
  / [`vllm`](../vllm/) / [`sglang`](../sglang/) /
  [`lmdeploy`](../lmdeploy/) /
  [`localai`](../localai/) /
  [`ramalama`](../ramalama/) /
  [`ollama`](../ollama/) are higher-level for that exact shape.
- You want a **proxy / gateway / router** in front of multiple
  LLM providers (rate limit, fallback, key vault) — that is
  [`litellm`](../litellm/) / [`portkey-gateway`](../portkey-gateway/)
  / [`helicone`](../helicone/), not BentoML.
- You don't have **Python in the serving stack** — BentoML is
  Python-first. Triton / KServe / Ray Serve fit non-Python
  inference better.
- You want **an inference server tightly coupled to PyTorch
  TorchScript** — that's TorchServe; BentoML is framework-agnostic
  and slightly heavier-weight.

## 6. Closest alternatives

- [`litserve`](../litserve/) — Lightning's "FastAPI on steroids
  for AI"; same shape but single-file and no build / deploy CLI.
  Pick when you don't need the bento packaging.
- [`openllm`](../openllm/) — BentoML's own LLM-specific front-end;
  zero-config OpenAI-compatible endpoint for the popular OSS LLMs.
- [`ray-serve`](https://docs.ray.io/en/latest/serve/) — Ray-native
  serving with strong multi-node / multi-model story; heavier
  install, better when you already run Ray.
- [`vllm`](../vllm/) / [`sglang`](../sglang/) /
  [`lmdeploy`](../lmdeploy/) — direct LLM inference engines you
  embed *inside* a BentoML service for the actual decode loop.
- [`modal`](../modal/) — alternative "Python function → cloud
  service" with a hosted-only (closed-source platform) model;
  BentoML is the BYO-cluster counterpart.

## 7. Hello-world

```python
# service.py
import bentoml
from transformers import pipeline

@bentoml.service(resources={"gpu": 1})
class Summariser:
    def __init__(self):
        self.pipe = pipeline("summarization", model="facebook/bart-large-cnn")

    @bentoml.api(batchable=True)
    def summarise(self, text: str) -> str:
        return self.pipe(text)[0]["summary_text"]
```

```bash
# Dev: hot-reload server on :3000 with OpenAPI at /docs
bentoml serve service:Summariser --reload

# Build a versioned bento
bentoml build

# Containerise → OCI image
bentoml containerize summariser:latest

# Deploy to BentoCloud (or K8s via the bentoctl plugin)
bentoml deploy summariser:latest
```

## 8. Killer feature, weakness, when to choose

- **Killer feature**: the *one* artefact (the bento) that pins
  Python + OS + CUDA + deps + service code + model refs, then
  runs unchanged from `bentoml serve` on a laptop to a multi-pod
  K8s deployment with adaptive batching and per-runner GPU shapes.
- **Weakness**: Python-only; non-trivial concept count (services /
  runners / bentos / models / IO descriptors); for a single-file
  LLM endpoint it is overkill vs. [`litserve`](../litserve/) or
  [`openllm`](../openllm/).
- **When to choose**: production ML / LLM serving where the team
  writes Python, you need reproducible builds + adaptive batching
  + multi-model topology + a clear path to managed cloud, and
  Triton / TorchServe feel too framework-specific.

## 9. Repo health (snapshot)

- Active development, weekly to bi-weekly releases on `main`;
  the 1.4 line landed end-2025 with a Rust-accelerated request
  router and stable Pydantic v2 IO. Latest **v1.4.38** on
  2026-04-02.
- Public surface (`@bentoml.service`, `@bentoml.api`,
  `bentoml.depends`, `bentoml.models`) has been stable across the
  1.4 line; pin to a minor version in production.
- Sister project [`openllm`](../openllm/) tracks BentoML and
  inherits its build / deploy semantics.
