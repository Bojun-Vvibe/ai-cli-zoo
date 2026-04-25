# ragflow

> Snapshot date: 2026-04. Upstream: <https://github.com/infiniflow/ragflow>

"**RAGFlow is a leading open-source Retrieval-Augmented
Generation (RAG) engine that fuses cutting-edge RAG with Agent
capabilities to create a superior context layer for LLMs.**"
RAGFlow from Infiniflow is the **batteries-included RAG
platform** end of the catalog: a Docker-Compose stack that
bundles a web UI, a document parser pipeline (DeepDoc — table /
figure / formula extraction with layout-aware OCR), a chunker
with template knobs per document class, a vector + full-text
hybrid retriever (Infinity / Elasticsearch), a re-ranker, an
agent / workflow builder, and an OpenAI-compatible chat /
HTTP API on top — all wired together, no glue code required.

## 1. Install footprint

- Primary install path is **Docker Compose**:
  `git clone https://github.com/infiniflow/ragflow && cd
  ragflow/docker && docker compose -f docker-compose.yml up -d`.
  Stack pulls ~5–10 GB of images (RAGFlow server, MySQL, Redis,
  MinIO, Elasticsearch / Infinity, the embedding model service).
  Default UI lands on <http://localhost:80>.
- GPU variants: `docker compose -f docker-compose-gpu.yml up -d`
  for CUDA-accelerated parsing / embedding.
- Slim "v0.x-slim" image is ~2 GB and skips bundled embedding
  models — point it at an external OpenAI / Ollama / vLLM
  endpoint instead.
- Python SDK: `pip install ragflow-sdk` for scripting against a
  running RAGFlow server (knowledge-base CRUD, chat, retrieval).
- API: REST + OpenAI-compatible `/v1/chat/completions` endpoint
  per chat-assistant for drop-in client usage.

## 2. Repo + version + license

- Repo: <https://github.com/infiniflow/ragflow>
- Latest release: **v0.25.0** (2026-04)
- License: **Apache-2.0** —
  <https://github.com/infiniflow/ragflow/blob/main/LICENSE>
- HEAD SHA: `fb95136f391fac8fa4288d4f687e473675c3cdb2`
- Default branch: `main`
- Language: Python (server), TypeScript (web UI)

## 3. Models supported

LLMs are **bring-your-own-endpoint**: configure the model
provider in the UI and point at OpenAI / Anthropic / Gemini /
DeepSeek / Mistral / Moonshot / Zhipu / Qwen-API / Tongyi /
Bedrock / Azure / Ollama / vLLM / LocalAI / Xinference / any
OpenAI-compatible URL. Embedding models: bundled
`BAAI/bge-large-zh-v1.5` and `BAAI/bge-large-en-v1.5` by
default; can swap to FlagEmbedding, Jina, Cohere, OpenAI,
Voyage, or any HF sentence-transformers model. Re-rankers:
`BAAI/bge-reranker-v2-m3`, Jina-rerank, Cohere-rerank.
Document parsers cover PDF (with table / figure layout),
DOCX, XLSX, PPTX, HTML, Markdown, JSON, CSV, EML, TXT, plus
images via OCR.

## 4. Simple usage

```bash
# 1. Boot the stack
git clone https://github.com/infiniflow/ragflow.git
cd ragflow/docker
docker compose -f docker-compose.yml up -d

# 2. Open http://localhost in a browser, register the first user
#    (becomes admin), add a model provider in Settings -> Model
#    Providers, then create a Knowledge Base and upload PDFs.

# 3. Or drive it from Python
pip install ragflow-sdk
```

```python
from ragflow_sdk import RAGFlow

rag = RAGFlow(api_key="ragflow-xxx", base_url="http://localhost")
kb = rag.create_dataset(name="handbook")
kb.upload_documents([{"name": "spec.pdf", "blob": open("spec.pdf","rb").read()}])
kb.async_parse_documents([d.id for d in kb.list_documents()])

assistant = rag.create_chat(name="Handbook Q&A", dataset_ids=[kb.id])
session = assistant.create_session()
for chunk in session.ask("What does section 4.2 say about retries?",
                         stream=True):
    print(chunk.content, end="", flush=True)
```

## 5. Why it's interesting

- **DeepDoc**, the in-tree document parser, is the standout
  feature: it does layout-aware PDF parsing (tables stay tables,
  figures get captions, formulas get LaTeX), which is the failure
  mode that kills naive `pypdf`-based RAG pipelines on real-world
  reports / specs / financial filings.
- **Hybrid retrieval out of the box** — vector + BM25 + re-rank,
  with the relative weight tunable per knowledge base, plus an
  optional knowledge-graph-based retrieval mode (entities +
  relations extracted at parse time).
- **Chunking templates per document class**: legal contract,
  research paper, presentation, manual, Q&A, table, resume,
  one-line, etc. — the chunker picks structure-aware boundaries
  rather than `RecursiveCharacterTextSplitter`-ing everything.
- **Visual citation traceback in the UI**: every chat answer
  links back to the highlighted region in the source PDF, which
  is what most "is this RAG actually grounded?" reviews want to
  see.
- **Agent / workflow builder** added in 0.20+ lets you string
  retrieval + tool-calls + branches into a flow, so RAGFlow
  becomes the agent host as well as the retriever.

## 6. Caveats

- Resource-hungry: the default Compose stack expects ~16 GB RAM
  and several CPU cores; the GPU variant for parsing wants more.
  Not a "laptop sidecar" RAG.
- Operationally, you are running MySQL + Redis + MinIO +
  Elasticsearch (or Infinity) + the RAGFlow server + an
  embedding model — that is real infrastructure to monitor and
  upgrade.
- Apache-2.0 — but watch the **commercial-use clause** in
  `LICENSE`/`NOTICE`: the project asks SaaS providers above a
  user threshold to obtain a commercial license. Read the
  `LICENSE` and `licenses/` directory before redistribution.
- The UI is the primary surface; the SDK / REST API works but
  trails the UI on feature parity in any given release.
- Bundled bge-large embeddings are good for English / Chinese;
  for highly specialised corpora (biomedical, code, legal) you
  will want to swap in a domain-tuned embedder.

## 7. When to choose

You want a **complete, hostable RAG product** with a UI, doc
parser, hybrid retrieval, re-ranker, citation UI, and chat /
HTTP API in one Compose file — and you'd rather operate one
opinionated platform than wire LangChain + Unstructured + a
vector DB + a re-ranker + a UI together yourself. Especially
strong when documents are **PDFs with real layout** (tables,
figures, formulas) where DeepDoc's parsing earns its keep.
Pair with a self-hosted [`vllm`](../vllm/) /
[`llama.cpp`](../llama.cpp/) / [`ollama`](../ollama/) endpoint
to keep generation on-prem; pair with
[`langfuse`](../langfuse/) or [`opik`](../opik/) for
observability across chat sessions. Skip in favour of
[`r2r`](../r2r/) if you want a similar batteries-included
RAG system with a Python-first surface and lighter ops; skip
in favour of [`haystack`](../haystack/) /
[`llama-index`](../llama-index/) /
[`langgraph`](../langgraph/) if you'd rather author your RAG
pipeline as code than configure a server; skip in favour of
[`chroma`](../chroma/) / [`qdrant`](../qdrant/) /
[`weaviate`](../weaviate/) if you only need the vector store
and will own everything around it.
