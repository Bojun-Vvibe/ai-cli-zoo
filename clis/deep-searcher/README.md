# deep-searcher

> Snapshot date: 2026-04-26. Upstream:
> <https://github.com/zilliztech/deep-searcher>

"**Open Source Deep Research Alternative to Reason and Search on
Private Data.**" deep-searcher is Zilliz's local-first alternative
to OpenAI / Gemini Deep Research: point it at a directory, a URL,
or a Milvus collection, give it a question, and it iteratively
plans → retrieves → reflects → re-queries → answers, with the
whole reasoning trace surfaced. The pipeline runs entirely against
your own LLM + vector store, so private corpora never leave the
host.

## 1. Install footprint

- CLI / lib: `pip install deepsearcher` exposes the `deepsearcher`
  Python module and a CLI entrypoint.
- Self-host: `git clone https://github.com/zilliztech/deep-searcher
  && pip install -e .` then `deepsearcher --query "..."`.
- Backend: bring your own Milvus / Zilliz Cloud (or local
  in-process Milvus Lite) for the vector layer.

## 2. Repo + version + license

- Repo: <https://github.com/zilliztech/deep-searcher>
- Latest release: rolling `master` (no tagged releases yet)
- License: **Apache-2.0** —
  <https://github.com/zilliztech/deep-searcher/blob/master/LICENSE>
- Default branch: `master`
- Language: Python
- Stars: ~7.7k

## 3. Models supported

LLM: OpenAI, DeepSeek, SiliconFlow, TogetherAI, Gemini, XAI,
Anthropic, Ollama, plus any OpenAI-compatible endpoint via the
`OpenAILLM` shim. Embedding: OpenAI, VoyageAI, SiliconFlow,
HuggingFace SentenceTransformers, and local Ollama embeddings.
Vector store: Milvus / Zilliz Cloud / Milvus Lite.

## 4. Notable angle

**Deep-research loop on private data with a configurable LLM and a
production-grade vector backend.** Where `gpt-researcher` and
`storm` target the open web, deep-searcher is built around private
corpora — PDF/MD/JSON/HTML loaders dump into Milvus and the agent
loop (RAGRouter → ChainOfRAG → Reflect → SubQueryGen) cites only
your indexed sources. Backed by the team that ships Milvus, so
the retrieval layer is the strongest piece of the stack, not an
afterthought.

## 5. Last verified

2026-04-26 via `gh api repos/zilliztech/deep-searcher`.
