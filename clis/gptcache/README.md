# gptcache

> Snapshot date: 2026-04. Upstream: <https://github.com/zilliztech/GPTCache>
> License file: <https://github.com/zilliztech/GPTCache/blob/main/LICENSE>
> Pinned: `0.1.44` (2024-08-01). Upstream is in **slow-maintenance
> mode** — last tagged release is 18+ months old, occasional fixes
> land on `main`. The architecture and APIs are stable enough that
> the staleness is "freeze it and use it" rather than "abandoned",
> but pin the exact PyPI version and do not expect new features.

A **semantic cache for LLM calls**. Drop-in proxy / library that
intercepts `openai.ChatCompletion.create(...)` (and equivalents),
embeds the request prompt, looks for a semantically-similar cached
response in the configured vector store, and returns the cached
response when similarity beats your threshold. On a miss it forwards
to the upstream LLM, embeds the prompt, and stores
`(embedding, request, response)` for next time.

The everyday surface is the Python SDK (`from gptcache.adapter import
openai` then call `openai.ChatCompletion.create(...)` exactly like
the SDK). LangChain and LlamaIndex adapters ship in-tree.

## 1. Install footprint

- `pip install gptcache` (or `uv pip install gptcache`). Pulls
  `numpy` and a tiny core; vector store + embedding deps are extras.
- Python 3.8.1+. No daemon; cache lives in process and persists to
  whichever scalar store + vector store you wire up.
- Workspace footprint: defaults to `sqlite + faiss` writing to
  `./sqlite.db` and `./faiss.index` in cwd — add both to `.gitignore`.
  Production installs swap to `postgres + milvus` /
  `mysql + qdrant` / `mongo + pgvector` / `redis (scalar) +
  redis-vector` via the `manager_factory("sqlite,faiss",
  vector_params={...})` builder.
- Embedding extras: `pip install gptcache[onnx]` for the bundled
  `paraphrase-albert-onnx` (small, fast, CPU-only, ~50 MB) — the
  default. Swap to `gptcache[sentence_transformers]` /
  OpenAI / Cohere / HuggingFace / FastText / data2vec / Rwkv via the
  `Embedding` adapters.

## 2. License

MIT (file is `LICENSE` at the repo root).

## 3. Models supported

GPTCache is **upstream-agnostic** by design — it caches at the
request / response boundary, not the provider boundary. Adapters
ship for:

- **OpenAI** — `from gptcache.adapter import openai` (drop-in for
  `ChatCompletion`, `Completion`, `Embedding`, `Image`, `Audio`).
- **LangChain** — `langchain.cache.GPTCache` (`set_llm_cache(...)`)
  caches every LLM in the chain.
- **LlamaIndex** — `gptcache.adapter.llama_index` wraps
  `LLMPredictor` / `ServiceContext`.
- **Replicate, Diffusers, Cohere, MiniGPT-4, StableLM, Dolly** —
  individual adapters.
- **Anything OpenAI-compatible** — point the OpenAI adapter at
  Ollama / vLLM / openllm / mlx-lm / LM Studio / LiteLLM /
  any provider-compat URL via `openai.api_base = "..."`. The cache
  doesn't know or care.

The **embedding model** is a separate axis (used to compute
similarity, never sent to the upstream LLM):
ONNX `paraphrase-albert` default (CPU, offline), HuggingFace
sentence-transformers, OpenAI `text-embedding-3-small` / `-large`,
Cohere, FastText, Rwkv, data2vec — pick by latency / quality /
egress trade-off.

## 4. MCP support

**No first-party MCP transport.** GPTCache predates MCP and sits one
layer below the protocol. Two integration shapes work today:

- Cache **inside an MCP server** — your MCP tool implementation
  calls `gptcache.adapter.openai.ChatCompletion.create(...)`; the
  cache transparently absorbs duplicate / similar tool invocations
  from the agent.
- Cache **in front of an MCP-aware client** — wire the OpenAI
  adapter at module-load and every model call from
  [`opencode`](../opencode/) / [`claude-code`](../claude-code/) /
  [`fast-agent`](../fast-agent/) (when configured for an
  OpenAI-compatible endpoint) gets transparent semantic caching.

GPTCache itself never exposes MCP — it has no resource / tool /
prompt surface to publish.

## 5. Sub-agent model

None — GPTCache is a side-effect layer, not a runtime. The cache
is shared across processes when you point them at the same
scalar + vector store, which is the substrate behaviour you want
when an agent fans out N parallel sub-agent calls and several
prompts collide.

For composite caching strategies, the `EvaluationStrategy` /
`PostProcessFunction` hooks let you chain "exact-match → semantic
similarity → LLM-rerank" inside one cache lookup, which is the
cache-layer counterpart to a small ranking pipeline.

## 6. Telemetry stance

**Off by default.** No analytics in the OSS library; the cache is
purely local-process by default and only egresses to (a) your
configured embedding model if you picked a remote one (OpenAI /
Cohere) and (b) the upstream LLM on cache misses.

The cached prompts and responses live in your scalar store
(SQLite / Postgres / Mongo / Redis) — treat the store like any
other LLM trace store and apply your normal retention policy.
The `Report.op_stat()` call dumps hit-rate / latency / cost-saved
counters as a local DataFrame; nothing leaves the process.

For sensitive workloads, pick the bundled ONNX
`paraphrase-albert-onnx` embedder and a local SQLite + FAISS pair
— the entire cache stack runs offline and the only network call
is to the upstream LLM on misses.

## 7. Prompt-cache strategy

This is **the entire point of the tool**. GPTCache implements the
strategy you'd otherwise write by hand:

- **Exact-match fast path** — string-equal prompt hash hits the
  scalar store directly, sub-millisecond.
- **Semantic similarity** — prompt embedding + ANN search over the
  vector store; configurable similarity threshold (`similarity =
  SearchDistanceEvaluation(max_distance=0.1)` for cosine).
- **Eviction** — LRU / LFU / FIFO over the scalar store, configurable
  `max_size`. Hot embeddings stay in memory; cold ones page out.
- **Post-processing** — `temperature` softmax over top-K matches if
  you want non-deterministic cache hits to mimic the upstream's
  sampling; deterministic top-1 by default.
- **Session scoping** — `Session(name="user-42")` namespaces the
  cache so per-tenant prompts don't collide; cache statistics
  bucket per session.

What it does **not** do: it does not speak Anthropic's prefix-cache
beta header or OpenAI's automatic prompt-caching protocol — those
are provider-side optimisations on the *miss* path. GPTCache is the
*hit* path that prevents the call in the first place.

## 8. Hot keybinds

No TUI, no daemon, no first-class CLI. Three Python surfaces:

```python
# 1. Drop-in OpenAI replacement
from gptcache import cache
from gptcache.adapter import openai
from gptcache.embedding import Onnx
from gptcache.manager import manager_factory
from gptcache.similarity_evaluation.distance import SearchDistanceEvaluation

onnx = Onnx()
data_manager = manager_factory(
    "sqlite,faiss",
    vector_params={"dimension": onnx.dimension},
)
cache.init(
    embedding_func=onnx.to_embeddings,
    data_manager=data_manager,
    similarity_evaluation=SearchDistanceEvaluation(),
)
cache.set_openai_key()

# Now this call hits the cache on round 2
resp = openai.ChatCompletion.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Who wrote Pride and Prejudice?"}],
)
```

```python
# 2. LangChain integration
from langchain.globals import set_llm_cache
from langchain.cache import GPTCache
import hashlib

def init_gptcache(cache_obj, llm: str):
    hashed = hashlib.sha256(llm.encode()).hexdigest()
    cache_obj.init(
        pre_embedding_func=lambda d, **_: d["prompt"],
        data_manager=manager_factory("sqlite,faiss",
                                     data_dir=f"map_cache_{hashed}"),
    )

set_llm_cache(GPTCache(init_gptcache))
```

```python
# 3. Inspect cache effectiveness
from gptcache.report import Report
print(Report().op_stat())  # hit_count, miss_count, latency, cost_saved
```

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Semantic cache as a one-import drop-in over
existing LLM SDK call sites**. The OpenAI adapter is a literal
namespace replacement (`from gptcache.adapter import openai`) so
you cache without touching any call site logic; the embedding +
scalar-store + vector-store + similarity-evaluator factory means
the same code runs as a process-local SQLite + FAISS pair on a
laptop and as a fleet-wide Postgres + Milvus pair in production
behind one config swap. On chatbot / RAG workloads with repetitive
question patterns, hit rates of 30–70% are realistic and translate
directly into latency cuts (ANN lookup is sub-100 ms vs. seconds
for a frontier model) and provider bill cuts of the same factor.

**Weakness.** **Slow-maintenance upstream + the inherent
correctness risk of semantic caching**. The 18-month gap since
`0.1.44` means newer providers (Anthropic Messages API tool-call
shape, OpenAI Responses API, Gemini 2.5 streaming) are not
first-class — you wire them via the OpenAI-compatible adapter and
hope. The semantic-cache premise itself is risky: a 0.1 cosine
threshold sometimes returns a cached answer to a *different*
question (negation, "not", numerical changes that the embedder
flattens) — pin a *tight* threshold (`max_distance=0.05` or
lower) for any output where being wrong matters, and run a
post-hit verification feedback function via
[`deepeval`](../deepeval/) / [`ragas`](../ragas/) /
[`trulens`](../trulens/) over a sample. Alternatives like
[Langfuse](https://github.com/langfuse/langfuse) prompt cache
or [LiteLLM](https://github.com/BerriAI/litellm)'s built-in cache
are more actively maintained but offer narrower semantic features.

**When to choose.**

- You have a **chatbot / RAG / agent workload** with high prompt
  repetition (FAQs, common how-tos, repeated retrieval queries)
  and want to **cut LLM bills + tail latency** without rewriting
  call sites.
- You want a **provider-agnostic cache layer** that sits above
  whichever LLM SDK / proxy you use, with a clean swap path
  between local-FAISS and clustered-Milvus as you scale.
- You need **per-tenant cache isolation** (`Session("tenant-X")`)
  and configurable similarity thresholds per route — the API
  surface for this is built in, not a bolt-on.

**When not to choose.** You only need **exact-match caching** —
LiteLLM proxy's built-in cache or a `functools.lru_cache` over the
prompt hash is simpler, no embedding model required. You need the
**latest provider features** (Responses API, Anthropic computer-use,
Gemini 2.5 flash thinking) — the upstream cadence won't keep up;
implement caching at the LiteLLM / your-proxy layer instead. You
need **strict deterministic correctness** (compliance, medical, legal,
finance numerical) — semantic caching is the wrong tool, every miss
must hit the source; use exact-match only or skip caching entirely.
You want an **AI gateway** with rate-limiting + key-rotation + audit
logging on top of caching — pick a real proxy (LiteLLM, Helicone,
Portkey) and use its built-in cache.
