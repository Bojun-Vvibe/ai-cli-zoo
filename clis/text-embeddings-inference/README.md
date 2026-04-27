# text-embeddings-inference

> Snapshot date: 2026-04. Upstream: <https://github.com/huggingface/text-embeddings-inference>
> Pinned release: `v1.9.3`. License file: `LICENSE` (sha `7d0e80345c787d5080678c89b3834b50e399c3f5`).

A Rust inference server (and CLI launcher) for text-embedding,
re-ranker, and sequence-classification models. Sibling project to
`text-generation-inference`, but tuned for the embedding side of a RAG
pipeline: tiny models, huge batches, sub-millisecond latency. The
default backend most modern vector DBs in this catalog (`qdrant`,
`weaviate`, `chroma`, `lancedb`) recommend pointing at when you don't
want to call OpenAI for every chunk.

## 1. Install footprint

- Docker is the supported path:
  `docker run -p 8080:80 ghcr.io/huggingface/text-embeddings-inference:1.9.3 --model-id BAAI/bge-large-en-v1.5`
- Bare-metal: `cargo install --git https://github.com/huggingface/text-embeddings-inference --rev v1.9.3 text-embeddings-router` (Rust 1.75+, optionally CUDA / Metal).
- The `text-embeddings-router` binary is the CLI; it both serves and
  exposes a `--help` for model loading, batching, and backend flags.
- Models are pulled from the Hugging Face Hub on first run; cached to
  `~/.cache/huggingface/`.

## 2. License

Apache-2.0.

## 3. Models supported

Embedding models: BGE family, GTE, E5, Jina, Nomic, Mixedbread,
Stella, Snowflake Arctic — anything with a BERT/RoBERTa/XLM-R/Mistral
backbone in the supported architectures list. Re-rankers: BGE-reranker,
Jina-reranker. Sequence classifiers: any HF `AutoModelForSequenceClassification`.
No generative LLMs — that's the other repo.

## 4. MCP support

**No.** Exposes an OpenAI-compatible `/v1/embeddings` endpoint plus
its own native HTTP API. MCP-aware clients can talk to it by setting
their embedding base URL, the same way they'd point at OpenAI.

## 5. Sub-agent model

N/A — this is an inference server, not an agent. The relevant
concurrency model is **dynamic batching**: incoming requests are
coalesced into batches sized to saturate the GPU/CPU, with
configurable max batch tokens and queue timeout.

## 6. Telemetry stance

Off by default. The router emits no analytics. The Hub fetch on first
model load is the only outbound call; pre-mount the model directory
to make it fully air-gapped.

## 7. Prompt-cache strategy

N/A for embeddings (every input is unique by definition). The
performance lever is **batching**, not caching. For re-rankers, no
cache either — you're scoring fresh (query, doc) pairs every call.

## 8. Hot keybinds (CLI / flags)

The CLI is `text-embeddings-router`; "keybinds" are launch flags:

| Flag | Effect |
|------|--------|
| `--model-id <hf-repo>` | Model to load |
| `--port 8080` | Listen port |
| `--max-batch-tokens N` | Cap on tokens per batch (latency vs throughput knob) |
| `--max-concurrent-requests N` | Backpressure threshold |
| `--dtype float16\|bfloat16` | Precision; halves memory on most cards |
| `--auto-truncate` | Silently truncate inputs over the model's max seq length |
| `--pooling cls\|mean\|splade` | Override pooling for models that support multiple |

## 9. Killer feature, weakness, when to choose

- **Killer:** the **dynamic batcher**. On a single L4 GPU it will
  saturate throughput at ~10k embeddings/sec for BGE-base, with p99
  latency under 30 ms — numbers that simply don't exist in
  pure-Python servers. The OpenAI-compatible endpoint means most
  catalog tools (`llm`, `chroma`, `qdrant`, `langchain`) just work
  by changing a base URL.
- **Weakness:** embeddings only. You still need a separate runtime
  (`vllm`, `sglang`, `ollama`, `llama.cpp`) for generation. Also,
  not every HF embedding architecture is supported — exotic models
  fall back to a "please use `sentence-transformers` directly" error.
- **Choose it when:** you're building a RAG system, you've outgrown
  calling OpenAI's embedding API for every chunk (cost or privacy),
  and you want the lowest-friction path to a self-hosted embedding
  endpoint your existing tooling already knows how to talk to.
