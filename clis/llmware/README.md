# llmware

> Snapshot date: 2026-04. Upstream: <https://github.com/llmware-ai/llmware>

"**Unified framework for building enterprise RAG pipelines with
small, specialised models.**" `llmware` is an opinionated end-to-end
RAG / agent toolkit centred on the bet that a fleet of **small
(1–8B), task-specialised models** — many published by the same
project as the *BLING*, *DRAGON*, *SLIM*, and *Industry-BERT*
families on Hugging Face — outperforms one giant general model for
contract analysis, financial-document QA, classification, NER,
SQL-from-text, and tool-call routing. The library bundles the
ingestion path (PDF / DOCX / PPTX / XLSX / HTML / MD parsers via a
bundled C-library shim), the chunker, the embedding store
(Postgres / Mongo / SQLite / Milvus / Qdrant / Chroma / Redis /
PG-vector / Neo4j / LanceDB / FAISS), the retriever, the prompt
catalog, and a `LLMfx` agent runner that chains those small models
into multi-step workflows.

## 1. Install footprint

- `pip install llmware` (Python ≥ 3.9). Pulls a curated set of deps
  (numpy, pydantic, requests, tokenizers, huggingface-hub,
  bundled native parsers as prebuilt shared libraries for macOS /
  Linux / Windows on x86-64 + arm64).
- Library-first: `from llmware.library import Library;
  from llmware.retrieval import Query;
  from llmware.prompts import Prompt;
  from llmware.agents import LLMfx`. No dedicated `llmware` CLI
  binary — operate from Python scripts / notebooks.
- Vector-store extras are *optional* — install `pymongo`,
  `psycopg2-binary`, `pymilvus`, `qdrant-client`, `chromadb`,
  `redis`, `neo4j`, `lancedb`, `faiss-cpu` only for the back-end you
  pick. SQLite + FAISS work zero-config for local prototyping.
- Optional: `llama-cpp-python` to run quantised GGUF locally; `onnx`
  / `onnxruntime` to run the ONNX-converted SLIM / BLING models;
  `openvino` for Intel-acceleration paths.

## 2. Repo + version + license

- Repo: <https://github.com/llmware-ai/llmware>
- Latest release: **v0.4.6** (2026-04-14)
- License: **Apache-2.0** —
  <https://github.com/llmware-ai/llmware/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (with bundled native parser binaries)

## 3. Models supported

- **Native fleet (HF org `llmware`)**: BLING (1–3B instruct, RAG-
  tuned), DRAGON (6–7B production RAG), SLIM (1–3B function-calling
  / classification / sentiment / NER / SQL / topic / tags / boolean /
  ratings), Industry-BERT (sector-specific embeddings),
  `bling-phi-3-gguf`, `dragon-yi-6b-gguf`, etc. Many ship in
  GGUF + ONNX + safetensors so the same workflow runs on
  llama.cpp / ONNX Runtime / Transformers.
- **External LLMs** via adapters: OpenAI, Anthropic, Cohere, Google
  (Gemini / Vertex), AWS Bedrock, Azure OpenAI, Ollama, vLLM, plus
  any OpenAI-compatible HTTP endpoint.
- **Embedding models**: the in-house Industry-BERT family,
  sentence-transformers, OpenAI / Cohere / Voyage / Jina embeddings.

## 4. When to use it

- You want a **batteries-included RAG stack** that owns ingestion →
  parsing → chunking → embedding → vector store → retrieval →
  prompt catalog → answer-with-evidence → fact-checker, instead of
  composing 6 libraries yourself. The `Library` / `Query` / `Prompt`
  triad is the whole surface.
- You explicitly want to **run small specialised models locally**
  for cost / privacy / latency reasons — function-calling on a 1B
  SLIM model is the headline benchmark for this project.
- You need **tabular + text + image** ingestion in one pipeline:
  the bundled parsers cover PDFs (with table extraction), Office
  docs, HTML, Markdown, and OCR via Tesseract.
- You want a **multi-vector-store abstraction**: prototype on SQLite
  + FAISS, swap to Postgres + pgvector or Milvus for production by
  changing one config line.

## 5. When NOT to use it

- You want a **thin provider-agnostic adapter** — llmware is a
  *framework*, not a one-line wrapper. Reach for
  [`aisuite`](../aisuite/), [`instructor`](../instructor/),
  [`mirascope`](../mirascope/), [`magentic`](../magentic/) instead.
- You want **declarative, optimisable prompt programs** with auto-
  prompt-tuning — that is [`dspy`](../dspy/)'s job; llmware's
  prompt catalog is template-shaped.
- You want a **prompt CLI from the shell** — llmware is library-only;
  use [`llm`](../llm/), [`aichat`](../aichat/),
  [`shell-gpt`](../shell-gpt/), [`mods`](../mods/) for that.
- You want **multi-agent orchestration with planner / executor /
  critic roles** — pick [`crewai`](../crewai/) /
  [`metagpt`](../metagpt/) / [`agno`](../agno/) /
  [`letta`](../letta/) instead. `LLMfx` is a *function-call router
  over small models*, not a full agent society.
- You want a **typed-output Pydantic story** — that is
  [`instructor`](../instructor/) / [`mirascope`](../mirascope/);
  llmware's outputs are dict-shaped with optional schema validators.

## 6. Closest alternatives

- [`txtai`](../txtai/) — embeddings-first RAG / search framework;
  lighter ingestion surface, similar "all-in-one" stance.
- [`db-gpt`](../db-gpt/) — RAG + agents oriented at databases /
  text-to-SQL; overlaps llmware's structured-data ambitions.
- [`crawl4ai`](../crawl4ai/) + [`docling`](../docling/) +
  [`marker`](../marker/) + a vector store + a prompt library =
  the à-la-carte recipe llmware bundles into one import.
- [`khoj`](../khoj/) — also self-hosted RAG over personal docs, but
  product-shaped (chat UI + scheduler) rather than framework-shaped.
- [`ragas`](../ragas/) — *evaluator* for RAG; pair with llmware to
  measure faithfulness / answer-relevancy / context-precision.

## 7. Repo health (snapshot)

- Steady release cadence on `main` — latest **v0.4.6** on
  2026-04-14; minor / patch releases land monthly.
- Notable companion artefacts on Hugging Face under the `llmware`
  org: 100+ small models in BLING / DRAGON / SLIM / Industry-BERT
  families, kept in step with library releases.
- Pre-1.0 — the public surface (`Library`, `Query`, `Prompt`,
  `LLMfx`) has been stable since the 0.3.x line but read the
  changelog before pinning long-term.
