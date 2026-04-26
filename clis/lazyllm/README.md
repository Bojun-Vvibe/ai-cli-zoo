# lazyllm

> Snapshot date: 2026-04-26. Upstream: <https://github.com/LazyAGI/LazyLLM>

"**A low-code development tool for building multi-agent LLMs
applications.**" LazyLLM is SenseTime / LazyAGI's batteries-included
framework for assembling multi-agent LLM apps from a unified
`Module` abstraction (online API, locally-hosted weights,
embedding, reranker, vector store, web UI), composing them with a
small data-flow DSL (`pipeline`, `parallel`, `switch`, `ifs`,
`loop`), and shipping the whole thing as a single one-click
deployment with a built-in lightweight gateway — POC-to-prod
without rewriting between development frameworks and serving
frameworks.

## 1. Install footprint

- Library: `pip install lazyllm` for the core flow + module
  layer (online providers + bring-your-own inference endpoint);
  `pip install lazyllm[full]` pulls inference / fine-tune /
  vector-store extras (lightllm, vllm, sentence-transformers,
  ChromaDB, Milvus client, etc.).
- CLI: `lazyllm run chatbot` boots an end-to-end chatbot web
  UI against an online provider (`LAZYLLM_OPENAI_API_KEY=…`); add
  `--model=internlm2-chat-7b` to swap to a `TrainableModule`
  backed by the local inference framework of your choice
  (lightllm or vllm), which LazyLLM auto-downloads + serves.
- Config: `~/.lazyllm/config.json` for keys + per-module
  defaults; `LAZYLLM_*` env vars override per shell.

## 2. Repo + version + license

- Repo: <https://github.com/LazyAGI/LazyLLM>
- Latest release: **v0.7.6** (published 2026-03-04)
- License: **Apache-2.0** —
  <https://github.com/LazyAGI/LazyLLM/blob/main/LICENSE>
- Default branch: `main`
- Language: Python
- Stars: ~3.8k

## 3. Models supported

- Online chat: OpenAI, Kimi (Moonshot), GLM (Zhipu), Qwen
  (DashScope), DeepSeek, Doubao, SenseNova, OpenRouter, plus any
  OpenAI-compatible endpoint via `OnlineChatModule(source=…,
  base_url=…)`.
- Local chat / fine-tune (`TrainableModule`): InternLM / InternLM2,
  Qwen / Qwen2 / Qwen2.5, Llama 2 / 3, Baichuan, ChatGLM, Mistral,
  Mixtral — auto-routed onto **lightllm** or **vllm** for serving
  and onto LLaMA-Factory / collie for fine-tuning, with
  multi-GPU + Slurm + bare-metal launchers handled by LazyLLM.
- Embeddings + reranker: BGE family, M3E, sentence-transformers,
  online providers (OpenAI / DashScope / Zhipu); vector stores:
  ChromaDB, Milvus, in-memory.
- Multimodal helpers: Stable-Diffusion / MusicGen / ChatTTS /
  SenseVoice modules ship as first-class `TrainableModule`
  variants for text→image, text→music, TTS, STT pipelines.

## 4. Notable angle

**One DSL spans "POC on an OpenAI key" and "self-hosted multi-GPU
fine-tune cluster" without rewriting the app.** Where
[`langgraph`](../langgraph/) and [`crewai`](../crewai/) own the
graph / role layer but punt inference + serving to whatever you
already have, and where [`vllm`](../vllm/) /
[`sglang`](../sglang/) own the serving layer but punt orchestration
to your application code, LazyLLM smashes the two halves into the
same module type: a `TrainableModule('internlm2-chat-7b')` is
literally swappable with an `OnlineChatModule(source='openai')`
inside the same `pipeline(...)`, and switching from "POC against
GPT-4o" to "self-hosted InternLM on 8×A100 served by vllm,
fine-tuned with LLaMA-Factory" is a one-line module swap — LazyLLM
handles the launcher (`SlurmLauncher`, `K8sLauncher`,
`EmpyrealLauncher`), the framework selection (lightllm vs vllm vs
HF-transformers based on the model card), and the gateway that
turns the whole multi-agent graph into one HTTP endpoint. The
trade-off: the abstraction is opinionated (you live inside
LazyLLM's module taxonomy), the China-origin online providers are
first-class while Anthropic / Bedrock / Vertex are second-class,
and the docs assume you are comfortable jumping between the
English and Chinese editions. Use it when you genuinely need the
"prototype → fine-tune → multi-GPU serve" pipeline as one tool;
keep [`langgraph`](../langgraph/) +
[`vllm`](../vllm/) separated when you only need orchestration and
already have a serving stack.

## 5. Last verified

2026-04-26 via `gh api repos/LazyAGI/LazyLLM` and
`gh api repos/LazyAGI/LazyLLM/releases/latest` → v0.7.6
(2026-03-04), license Apache-2.0 at
<https://github.com/LazyAGI/LazyLLM/blob/main/LICENSE>.
