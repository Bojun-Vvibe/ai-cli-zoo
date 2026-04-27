# web-llm

> Snapshot date: 2026-04. Upstream: <https://github.com/mlc-ai/web-llm>
> Pinned release: `v0.2.83` (HEAD `8a36ea0678f7d644c8d9263341de70195bfcdd59`).
> License file: [`LICENSE`](https://github.com/mlc-ai/web-llm/blob/main/LICENSE)
> (sha `7d10a85f06eeca6d5b96769e1941b3d55109c055`, Apache-2.0).

`web-llm` is the MLC team's in-browser LLM inference engine: a
TypeScript / WebAssembly package that loads a quantised model
(Llama 3, Qwen, Phi, Gemma, Mistral, DeepSeek, etc.) into a
WebGPU pipeline and exposes it behind an OpenAI-compatible
JavaScript SDK. The "CLI surface" in scope here is the toolchain
side — `npx mlc_llm` / the bundled `@mlc-ai/web-llm` package
scripts that fetch / pin / convert prebuilt model libraries and
wire them into a static site or Service Worker. It is the
catalog's reference for **a zero-server local LLM runtime**: the
inference loop runs in the user's browser tab, no daemon, no
Docker, no `localhost:11434`.

## 1. Install footprint

- `npm install @mlc-ai/web-llm` (or `yarn add` / `pnpm add`);
  also loadable from a CDN as
  `https://esm.run/@mlc-ai/web-llm`.
- Hard prereq: a browser with WebGPU enabled — Chrome 113+ /
  Edge 113+ / Safari 18+ on macOS / Chrome on Android. Firefox
  Nightly behind a flag.
- Models are downloaded from the `mlc-ai` HuggingFace org on
  first use and cached in the browser's `OPFS` / `IndexedDB`;
  typical 7B-int4 ≈ 3.5 GB, Phi-3-mini-int4 ≈ 1.8 GB.
- Toolchain side: `pip install mlc-llm` provides the `mlc_llm`
  CLI for converting / quantising new checkpoints into the
  `.wasm` + shard format the JS engine consumes.

## 2. Repo, version, license

- Repo: <https://github.com/mlc-ai/web-llm>
- Latest release: `v0.2.83` (2026-04-24).
- HEAD pinned at this snapshot:
  `8a36ea0678f7d644c8d9263341de70195bfcdd59`.
- License: Apache-2.0. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/mlc-ai/web-llm/blob/main/LICENSE)
  (sha `7d10a85f06eeca6d5b96769e1941b3d55109c055`).

## 3. What it actually does

The runtime is a JS class (`MLCEngine` / `WebWorkerMLCEngine` /
`ServiceWorkerMLCEngine`) that mirrors the OpenAI Node SDK
shape:

```ts
import { CreateMLCEngine } from "@mlc-ai/web-llm";
const engine = await CreateMLCEngine("Llama-3.1-8B-Instruct-q4f16_1-MLC");
const reply = await engine.chat.completions.create({
  messages: [{ role: "user", content: "hi" }],
  stream: true,
});
```

Streaming, JSON mode (with arbitrary JSON schema enforced by the
WebAssembly grammar matcher), seeded sampling, logit bias, and
parallel function-calling are supported. The
`WebWorkerMLCEngine` flavour offloads the inference loop to a
dedicated worker so the main thread keeps painting at 60 fps;
the `ServiceWorkerMLCEngine` flavour persists the loaded model
across page reloads. A Chrome-extension scaffold ships in
`examples/chrome-extension/`.

## 4. MCP support

None. The runtime exposes the OpenAI Chat Completions / JSON-
mode surface only. MCP-style tool routing has to live in the
host page, calling the engine for the LLM step.

## 5. Sub-agent model

None at the runtime layer. Function-calling lets the host page
implement an agent loop on top, but `web-llm` itself is a single
inference engine instance per page.

## 6. Telemetry stance

Off / not applicable. The engine runs entirely client-side; the
only network traffic is the initial model-shard download from
HuggingFace (or whatever CDN the host page configures via
`appConfig.model_list`). No analytics, no phone-home, no
provider key — there is no provider.

## 7. Token / context strategy

Per-model, baked into the converted `.wasm` library. Default
context lengths track the upstream model (8k for Llama-3.1
small variants, 32k for Qwen3-4B, 128k for some Phi-3 builds);
the host page can override with `context_window_size` /
`sliding_window_size` at engine creation time, bounded by
available WebGPU memory. KV-cache lives in the GPU buffer of
the active tab; closing the tab frees it.

## 8. Hot keybinds

None — it is a JS library, not an interactive CLI. The
companion [WebLLM Chat](https://chat.webllm.ai/) reference UI
uses standard chat-app affordances (`Enter` to send, `Shift-
Enter` for newline).

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A real OpenAI-compatible LLM endpoint
that lives inside the user's browser tab, with WebGPU-
accelerated decode and zero server cost.** Static-site
deployments (a single `index.html` on GitHub Pages, an
Electron / Tauri shell, a Chrome extension) get a private
local model with no Docker, no Ollama daemon, no API key, and
no per-token billing — the user's GPU pays. Quantised 7-8B
models hit double-digit tok/s on a recent Mac / Windows
discrete GPU; Phi-3-mini and Gemma-2B clear 30 tok/s on
integrated Intel / Apple GPUs.

**Weakness.** Cold-start is the multi-gigabyte model download;
WebGPU coverage on Linux and Firefox is still patchy; Safari
mobile is gated by per-tab memory limits that exclude anything
above ~3B parameters; the engine is a JS library, not a TUI /
shell CLI, so the rest of this catalog cannot point
`OPENAI_BASE_URL` at it without writing a tiny relay
(`fastify` route → `MLCEngine`) themselves.

**Choose `web-llm` when** the deployment surface is a webpage,
a browser extension, or an Electron / Tauri app and the goal
is a private LLM with zero server infra. **Choose something
else when** the consumer is a terminal CLI or a shell pipeline
([`ollama`](../ollama/), [`lms`](../lms/),
[`ramalama`](../ramalama/), [`mlx-lm`](../mlx-lm/)), the
target hardware is GPU-less ([`llamafile`](../llamafile/),
[`llama.cpp`](../llama.cpp/)), or batched throughput across
many concurrent users matters
([`vllm`](../vllm/), [`sglang`](../sglang/),
[`truss`](../truss/)).
