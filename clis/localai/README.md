# localai

> Snapshot date: 2026-04. Upstream: <https://github.com/mudler/LocalAI>
> Latest release at snapshot: **v4.1.3** (2026-04-06).
> License file: [`LICENSE`](https://github.com/mudler/LocalAI/blob/master/LICENSE) — MIT.

A drop-in, OpenAI-compatible REST API that runs everything **locally**:
chat completions, embeddings, image generation (Stable Diffusion /
Flux), text-to-speech, speech-to-text (Whisper), and reranking — all
behind the same `/v1/...` routes the OpenAI SDK already speaks. No
GPU required; no cloud account required. The headline pitch is:
"swap `https://api.openai.com` for `http://localhost:8080` in your
existing client and keep going."

This entry exists in the catalog because most other "local LLM" tools
(`ollama`, `ramalama`, `llamafile`) cover **only chat completions**.
`LocalAI` is the broader **OpenAI-API surface**: if your existing app
also calls `/v1/embeddings`, `/v1/audio/transcriptions`,
`/v1/images/generations`, or `/v1/rerank`, this is the one server that
covers all of them under one process.

## 1. Install footprint

- One binary or one container image. `curl https://localai.io/install.sh
  | sh`, `brew install localai`, or `docker run -p 8080:8080
  localai/localai:latest-cpu`.
- Backends are pulled lazily as separate "model gallery" artifacts
  (`llama.cpp`, `whisper.cpp`, `stable-diffusion.cpp`, `bark`, `piper`,
  `rerankers`, `vLLM`, `transformers`, `exllama2`, `mamba`, `tinydream`,
  `coqui`, etc.); a fresh install is ~150 MB before you pull a model.
- Per-user model store at `~/.local/share/localai/models/` (or
  `/build/models` inside the container). Config via YAML files per
  model in the model directory.
- HTTP server on `:8080` by default. No daemon outside that process.

## 2. License

MIT (see [`LICENSE`](https://github.com/mudler/LocalAI/blob/master/LICENSE)).
Bundled backends carry their own licenses (MIT for `llama.cpp` and
`whisper.cpp`, CreativeML Open RAIL-M for Stable Diffusion weights,
etc.) — read each model card before redistributing.

## 3. Models supported

Multi-modality, all local:

- **Chat / instruct:** any GGUF (Llama 3.x, Qwen 2.5 / 2.5-Coder, Gemma
  2/3, Mistral, Mixtral, Phi-3.x, DeepSeek-V3 / R1, Granite-Code,
  Hermes, Nous, Yi), plus vLLM / transformers / exllama2 backends for
  full-precision GPU inference.
- **Embeddings:** sentence-transformers, BERT, `bge-*`, `e5-*`, OpenAI-
  compatible vector outputs.
- **Image generation:** Stable Diffusion 1.5 / 2.x / XL / 3, Flux,
  PixArt — via `stable-diffusion.cpp` or diffusers backends.
- **Speech-to-text:** any Whisper checkpoint via `whisper.cpp`.
- **Text-to-speech:** Bark, Piper, Coqui XTTS, Parler-TTS.
- **Reranking:** `bge-reranker-*`, `jina-reranker-*`.
- **No closed-weight cloud models** by design.

## 4. MCP support

**No.** `LocalAI` is a serving layer; MCP belongs to the agent client.
Any MCP-aware CLI in this catalog points at the local OpenAI-compatible
endpoint and handles MCP itself.

## 5. Sub-agent model

None — serving layer, not an agent.

## 6. Telemetry stance

**Off. No phone-home at all.** The server does not ship analytics; the
only outbound network traffic is whatever you point a model gallery
fetch at (Hugging Face, OCI registries, your own mirrors).

## 7. Prompt-cache strategy

Inherits each backend's native cache: `llama.cpp` KV cache for chat,
diffusers UNet caching for images, Whisper's encoder reuse for batched
transcription. No provider-side prefix caching — there is no provider.

## 8. Hot keybinds

There is no TUI. The interface is HTTP. Three convenient surfaces:

| Surface | Invocation | Purpose |
|---------|------------|---------|
| HTTP API | `POST :8080/v1/chat/completions` etc. | OpenAI-compatible endpoint |
| Web UI | `http://127.0.0.1:8080` | Browser chat / model gallery installer |
| `local-ai` CLI | `local-ai run llama-3.2-3b-instruct` | One-shot model fetch + serve |

## 9. Killer feature, weakness, when to choose

- **Killer:** **one server, the whole OpenAI surface.** Chat,
  embeddings, transcription, image generation, TTS, and reranking all
  speak the same `/v1/...` shape an existing OpenAI client already
  knows. You can `OPENAI_API_BASE=http://localhost:8080/v1` an entire
  app off the cloud in one env var without touching code.
- **Weakness:** **a lot of moving parts.** Each modality is a separate
  backend with its own quant formats, accelerator quirks, and YAML
  schema. Compared to `ollama` (chat-only, very smooth) or `llamafile`
  (one file, chat-only), `LocalAI` is meaningfully more configuration
  surface in exchange for the broader API.
- **Choose it when:** your existing application already calls more than
  one OpenAI endpoint family (chat **and** embeddings, or chat **and**
  Whisper) and you want to point all of it at a single local server,
  with no per-modality glue and no cloud egress.

## Pitfalls observed in real use

1. **Model YAML schema drifts between minor versions.** A `model.yaml`
   that worked on v3.x may need a one-line update on v4.x (`backend:`
   names changed, template parameters renamed). Pin the container tag
   in production; do not float on `latest`.
2. **`latest` defaults to CPU.** Use `latest-gpu-nvidia-cuda-12`,
   `latest-aio-gpu-hipblas`, or the explicit `metal` tag if you want
   GPU acceleration; otherwise the server happily runs everything on
   CPU and you wonder why a 70B model is slow.
3. **Model gallery downloads are large and not resumable across
   container restarts.** Pull models into a mounted volume
   (`-v $PWD/models:/build/models`) before you run anything important.
4. **OpenAI client compatibility is "almost".** A handful of fields
   (`logprobs`, structured `tool_choice` shapes, vision content blocks
   for some backends) silently lose information; smoke-test your exact
   request shape before swapping out the base URL in production.
5. **No built-in auth.** The server listens unauthenticated; put it
   behind a reverse proxy with HTTP basic auth, mTLS, or bind it to
   `127.0.0.1` only — do **not** expose `:8080` to the internet.
