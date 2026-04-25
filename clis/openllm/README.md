# openllm

> Snapshot date: 2026-04. Upstream: <https://github.com/bentoml/OpenLLM>

"**Self-Hosting LLMs Made Easy.**" `openllm` is a single CLI from
the BentoML team that turns *any* open-weights LLM into an OpenAI-
compatible HTTP endpoint with one command (`openllm serve llama3.3:70b`),
plus a built-in chat UI at `/chat`, a model repository CLI for adding
custom weights, and a one-shot `openllm deploy` path that pushes the
same model to BentoCloud (or a self-hosted BentoML cluster) with the
configured GPU shape, autoscaling, and Docker image baked in. The
CLI is the front door; the heavy lifting is delegated to vLLM /
Transformers / SGLang / llama.cpp under the hood depending on the
model bento.

## 1. Install footprint

- `pip install openllm` (Python ≥ 3.9). Pulls `bentoml`, `pydantic`,
  `httpx`, `typer`, `rich`. Backend frameworks (`vllm`, `transformers`,
  `sglang`, `llama-cpp-python`) install lazily on first model serve.
- `openllm hello` is the canonical onboarding (interactive picker:
  shows model catalog → required GPU → starts serve).
- Linux + macOS supported; GPU serving wants Linux + CUDA 12.x for
  the vLLM path, MPS works on Mac for smaller models via the
  Transformers backend.
- Optional: a Docker daemon for the `openllm build` / `openllm deploy`
  containerisation path; a BentoCloud account for managed deploy.

## 2. Repo + version + license

- Repo: <https://github.com/bentoml/OpenLLM>
- Latest release: **v0.6.30** (2025-04-21)
- License: **Apache-2.0** —
  <https://github.com/bentoml/OpenLLM/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Curated catalog (see `openllm model list`): DeepSeek (R1 671B,
V3, Coder), Llama 3.1 / 3.2 / 3.3 / 4 (8B → 70B → 17B16E MoE),
Qwen 2.5 / 3 (incl. Coder, VL), Mistral / Mistral-Large / Mixtral,
Gemma 2 / 3, Phi 3 / 4, Pixtral, Jamba 1.5, plus community
additions. Each entry is a *bento* — a versioned manifest that
pins the backend (vLLM / Transformers / SGLang), the dtype, the
chat template, and the required GPU shape, so `openllm serve
llama3.3:70b` is reproducible across machines. Custom models go in
via `openllm repo add <name> <git-url>` pointing at a repo of
bento manifests. The exposed surface is OpenAI-compatible
`/v1/chat/completions` + `/v1/completions` + `/v1/models` with
streaming, so any OpenAI client SDK or any other CLI in this
catalog can target it unchanged.

## 4. MCP support

No (server). The model endpoint is plain OpenAI-compatible HTTP;
for MCP tooling, point an MCP-aware client
([`opencode`](../opencode/), [`codex`](../codex/),
[`fast-agent`](../fast-agent/), [`continue`](../continue/)) at the
`openllm` endpoint as the model backend and let *that* client
provide MCP. The library design does not embed an agent loop, so
adding MCP server-side would mean writing your own BentoML
service.

## 5. Sub-agent model

None. `openllm` is a model server, not an agent framework. One
process per loaded model; concurrency is per-request continuous
batching inside the chosen backend (vLLM excels here). Multi-model
fleets are expressed at the orchestration layer (Kubernetes
Deployment per bento, or BentoCloud's managed scaler), not inside
the CLI.

## 6. Telemetry stance

Off by default in OSS. BentoML's tracking (when enabled) is opt-in
via `BENTOML_DO_NOT_TRACK=False` reversal — the default in CI
images is `BENTOML_DO_NOT_TRACK=True`. The hosted BentoCloud control
plane is opt-in via `bentoml cloud login`. The serve binary itself
binds to `0.0.0.0:3000` by default — flip it to `127.0.0.1` if
you don't want LAN exposure during local dev.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **`openllm hello` → pick a model → typed
chat at `localhost:3000/chat` in three minutes, with the *same*
manifest reproducible on a 16-GPU box.** The bento abstraction is
the trick: a model entry pins the backend, dtype, chat template,
GPU shape, and serving config in one YAML, so the dev-laptop
`openllm serve gemma3:3b` and the production `openllm deploy
llama3.3:70b --gpu A100:80GB:2` use the same definition format —
you don't reimplement serving config when you scale. The catalog
covers the models people actually want (including DeepSeek-R1 671B,
Llama 4 17B16E MoE, Qwen3-Coder) and the vLLM backend gets you
production-grade throughput (continuous batching, paged attention,
prefix caching) without you assembling vLLM by hand. The chat UI
at `/chat` is genuinely usable for spot-checks and demos — it
beats curl-ing `/v1/chat/completions`.

**Weakness.** **It's a deployment tool, not a development tool —
no agent loop, no editor surface, no MCP.** If you want to *use*
an LLM to edit code, `openllm` is not what you reach for; it's
what you point [`opencode`](../opencode/) /
[`continue`](../continue/) / [`aider`](../aider/) at. The Linux+
CUDA bias is real: the vLLM backend (the high-throughput path) is
NVIDIA-only, and Mac users get the slower Transformers backend
that doesn't match [`mlx-lm`](../mlx-lm/) on the same hardware —
on Apple Silicon, prefer `mlx_lm.server`. Release cadence has
slowed (v0.6.30 in 2025-04, last commit 2025-09); the project is
maintained but not racing — the BentoML team's energy is on the
broader BentoML SDK, not on the `openllm` CLI specifically. Adding
a model that isn't in the catalog and isn't a HuggingFace
checkpoint with a stock chat template means writing a bento
manifest.

**When to choose.** You have NVIDIA GPUs (one box or a cluster),
you want a *production-shaped* OpenAI-compatible endpoint with one
command, and you want the same manifest to ship to a managed
deployment without rewriting it. Pair with [`continue`](../continue/)
or [`fast-agent`](../fast-agent/) on the client side, with
[`promptfoo`](../promptfoo/) for endpoint regression tests, and
with `vllm` directly if you outgrow the bento abstraction. Skip
in favour of [`mlx-lm`](../mlx-lm/) if you're Apple-Silicon only
(faster, lighter, same OpenAI surface), [`ollama`](../ollama/) if
you want a single static binary across Linux/Mac/Windows for a
small team without GPU concerns, [`localai`](../localai/) for the
"OpenAI-compatible Swiss-army knife" with embeddings + image-gen +
TTS in one process, or [`llamafile`](../llamafile/) for a literal
single executable you can email someone.
