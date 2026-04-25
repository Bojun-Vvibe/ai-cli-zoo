# ogx

> Snapshot date: 2026-04. Upstream: <https://github.com/ogx-ai/ogx>
> (formerly `meta-llama/llama-stack` — repo was renamed and the org
> migrated to `ogx-ai`; both URLs still resolve via GitHub redirect.)

"**Open GenAI Stack.**" `ogx` is the rebranded successor to
Meta's `llama-stack` project: an **opinionated, API-first
runtime** that defines a fixed set of REST APIs (Inference,
Safety, Agents, Memory, RAG, Eval, Telemetry, Post-Training,
Tool Runtime) and lets you mount **provider implementations**
under each API — so a single `ogx run` process gives you a
uniform `/v1/inference/chat-completion`, `/v1/agents`,
`/v1/safety/run-shield`, `/v1/memory/insert` surface whether
the actual backends are local llama.cpp, remote Together /
Fireworks / Bedrock / Ollama / vLLM / Cerebras, on-device
PyTorch, or a hosted Llama API. The CLI ships **stack
distributions** (preset combinations of providers) so you can
go from `ogx build` → `ogx run` to a working multi-provider
endpoint in two commands.

## 1. Install footprint

- `pip install ogx` (formerly `pip install llama-stack`); the
  pip name has migrated, the legacy alias still installs the
  same package for now.
- One CLI: `ogx build` (assemble a distribution YAML +
  Conda/venv/Docker image), `ogx run <distribution>` (serve the
  REST APIs), `ogx model list / download / verify`, `ogx
  configure`, `ogx providers list`, plus a Python client SDK
  (`ogx-client`) that talks to the same APIs.
- Python ≥ 3.10. Works on Linux / macOS (including Apple
  Silicon for the local-inference distros) / Windows; Docker
  images published per stack distribution.

## 2. Repo + version + license

- Repo: <https://github.com/ogx-ai/ogx>
- Latest release: **v0.7.1** (2026-04-08)
- License: **MIT** —
  <https://github.com/ogx-ai/ogx/blob/main/LICENSE>
- HEAD SHA: `379d0720c743b651ccf33130591759c2f7fbdec2`
- Default branch: `main`
- Language: Python (~8k stars)

## 3. Models / providers supported

**Inference providers** plug into one API: Meta Reference (local
PyTorch), Llama API, Ollama, vLLM, llama.cpp / llamafile, MLX
(Apple Silicon), Together, Fireworks, Cerebras, Bedrock, Groq,
SambaNova, NVIDIA NIM, Watsonx, plus any OpenAI-compatible
endpoint via the generic provider. **Safety**: Llama Guard 3 /
4, Prompt Guard, custom shields. **Memory / RAG**: FAISS,
Chroma, Weaviate, Qdrant, PGVector, Milvus, SQLite-Vec.
**Tool Runtime**: built-in web search (Brave, Tavily, Bing),
code interpreter, MCP-server tool sources. Models that ride
the Inference API: any Llama 3.x / 4 variant first-class, plus
any HF / Ollama / vLLM model the chosen provider can load.

## 4. Simple usage

```bash
# 1. list the preset stack distributions
ogx distributions list
# -> ollama, fireworks, together, meta-reference-gpu, ...

# 2. build a local-inference stack (Ollama + FAISS + Llama Guard)
ogx build --distribution ollama --image-type venv

# 3. run it — exposes REST APIs on :5001
ogx run ~/.ogx/distributions/ollama/ollama-run.yaml

# 4. hit the uniform Inference API from any HTTP client
curl http://localhost:5001/v1/inference/chat-completion \
  -H 'Content-Type: application/json' \
  -d '{
        "model_id": "llama3.2:3b",
        "messages": [{"role":"user","content":"hello"}]
      }'

# 5. or use the Python client against the same endpoint
python - <<'PY'
from ogx_client import OgxClient
c = OgxClient(base_url="http://localhost:5001")
print(c.inference.chat_completion(
    model_id="llama3.2:3b",
    messages=[{"role":"user","content":"hello"}],
).completion_message.content)
PY
```

## 5. Why it's interesting

- **API surface is the contract, not the model** — code written
  against `/v1/inference/chat-completion` keeps working when
  the underlying provider swaps from Ollama to Together to
  Bedrock; same `/v1/agents`, `/v1/safety`, `/v1/memory` story.
- **Stack distributions are the deployable unit**: a YAML
  pinning one provider per API plus their config; `ogx build`
  freezes that into a venv / Conda env / Docker image, `ogx run`
  serves it. Re-deploying someone else's stack is "share the
  YAML, run two commands."
- **Safety + RAG + Tools + Eval are part of the same stack**,
  not bolted on — Llama Guard runs as a first-class API endpoint,
  vector stores are providers under `/v1/memory`, and the agent
  loop in `/v1/agents` already knows how to call them all.
- **Multi-environment story is built-in**: same client code
  targets local-laptop (`distribution=ollama`), single-GPU
  (`meta-reference-gpu`), and multi-tenant cloud
  (`together` / `fireworks` / `bedrock`) — useful for "develop
  on Mac, deploy to cluster" loops.

## 6. Caveats

- **Active rebrand**: package + repo recently migrated from
  `llama-stack` to `ogx`; some docs, examples, and downstream
  blog posts still reference the old names. Pin a version when
  citing APIs.
- The API spec is opinionated — if your serving setup already
  speaks raw OpenAI / Anthropic / Bedrock, ogx adds a layer
  rather than removes one. The win shows up at 3+ providers
  or 3+ APIs (inference + safety + RAG).
- Provider coverage is uneven — **Inference** has the most
  backends; **Post-Training** and **Eval** APIs are still
  filling in providers, so check the matrix in the repo before
  betting on a non-obvious combination.
- Local heavy distributions (`meta-reference-gpu`) need the
  matching CUDA + torch toolchain; the lightweight
  `ollama` / `together` distros are the friendlier on-ramps.
