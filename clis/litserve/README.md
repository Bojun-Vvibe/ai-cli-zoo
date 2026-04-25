# litserve

> Snapshot date: 2026-04. Upstream: <https://github.com/Lightning-AI/LitServe>

"**A minimal Python framework for building custom AI inference
servers with full control over logic, batching, and scaling.**"
LitServe from the Lightning AI team is a thin, FastAPI-based
serving layer where you write a `LitAPI` subclass with
`setup()` / `decode_request()` / `predict()` / `encode_response()`
methods, hand it to `LitServer(...).run()`, and get a production
HTTP endpoint with **dynamic batching, GPU streaming, multi-worker
autoscaling, OpenAI-compatible spec, and OpenTelemetry traces** for
free. Unlike monolithic LLM servers (vLLM, TGI), LitServe is
model-agnostic: same surface serves a Llama-3 chat endpoint, a
Whisper transcription endpoint, an embedding endpoint, and a
classical sklearn classifier behind one process.

## 1. Install footprint

- `pip install litserve` — pure Python, ~2 MB; pulls FastAPI +
  uvicorn + pydantic.
- Optional: `pip install 'litserve[all]'` for OpenAI-spec helpers
  and OTel exporters.
- No CLI binary; the entry point is `python server.py` where
  `server.py` ends in `LitServer(api).run(port=8000)`. The
  package does ship one helper, `litserve.test_examples`, for
  scaffolding starter files.
- Python ≥ 3.9. Runs anywhere FastAPI runs (Linux / macOS /
  Windows / Docker / Kubernetes); GPU optional, picked up via
  `accelerator="cuda"` / `"mps"` / `"auto"`.

## 2. Repo + version + license

- Repo: <https://github.com/Lightning-AI/LitServe>
- Latest release: **v0.2.17** (2025-12-23)
- License: **Apache-2.0** —
  <https://github.com/Lightning-AI/LitServe/blob/main/LICENSE>
- HEAD SHA: `a69b6354f634f636298daba2637d68075bf5b681`
- Default branch: `main`
- Language: Python

## 3. Models supported

Anything with a Python inference call. First-class examples in
the repo cover Llama / Mistral / Qwen via `transformers`, vLLM,
or llama-cpp-python; Whisper, Stable Diffusion, ControlNet,
SAM, BERT embeddings, sklearn / xgboost classifiers, and
multi-modal pipelines (audio → text → image). Compound
endpoints — one request that fans out across several models in
`predict()` — are the showcase use case.

## 4. Simple usage

```python
# server.py
import litserve as ls
from transformers import pipeline

class SentimentAPI(ls.LitAPI):
    def setup(self, device):
        self.pipe = pipeline("sentiment-analysis", device=device)

    def decode_request(self, request):
        return request["text"]

    def predict(self, text):
        return self.pipe(text)[0]

    def encode_response(self, output):
        return {"label": output["label"], "score": output["score"]}

if __name__ == "__main__":
    server = ls.LitServer(
        SentimentAPI(),
        accelerator="auto",
        max_batch_size=16,
        batch_timeout=0.05,
        workers_per_device=2,
    )
    server.run(port=8000)
```

```bash
python server.py
curl -X POST http://localhost:8000/predict \
  -H 'Content-Type: application/json' \
  -d '{"text":"litserve is surprisingly nice"}'
```

## 5. Why it's interesting

- **Custom logic is the point**, not a side door — you own
  `predict()`, so compound pipelines (RAG with a re-ranker,
  Whisper → translate → TTS, vision → caption → embed) are
  one class instead of a service mesh.
- **Dynamic batching + multi-worker autoscaling come free**:
  `max_batch_size` / `batch_timeout` / `workers_per_device`
  are constructor args; LitServer assembles the request queue,
  GPU pinning, and back-pressure without a separate scheduler.
- **OpenAI-compatible spec is one mixin away** (`LitAPI` →
  `OpenAISpec`), so a custom `predict()` becomes a drop-in
  `/v1/chat/completions` endpoint that any OpenAI client speaks
  to without translation.
- **Streaming + async + GPU offload** are first-class: `yield`
  from `predict()` for SSE token streaming, `async def
  predict()` for I/O-bound endpoints, `LitServer(...,
  accelerator="cuda", devices="auto", workers_per_device=2)` to
  saturate multi-GPU boxes.

## 6. Caveats

- LitServe is the *framework*, not a model registry — you write
  the Python and bring the weights; no `lit serve llama3` shortcut.
- For pure single-model LLM serving with the highest possible
  tokens/sec on one architecture, dedicated engines (vLLM,
  lmdeploy, sglang) still win on raw throughput; LitServe wins
  when the endpoint is *not* one HF chat model.
- No built-in auth or rate-limiting — front it with a reverse
  proxy (Caddy, nginx, an API gateway) before exposing publicly.
- The "no CLI" surface means deployment is `python server.py` +
  your own process manager (systemd, Docker, k8s); there is no
  `litserve start` daemon.
