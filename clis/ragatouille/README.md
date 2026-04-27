# ragatouille

> Snapshot date: 2026-04. Upstream: <https://github.com/AnswerDotAI/RAGatouille>
> Pinned release: `0.0.9` (HEAD `e75b8a964a870dea886548f78da1900804749040`).
> License file: [`LICENSE`](https://github.com/AnswerDotAI/RAGatouille/blob/main/LICENSE)
> (sha `f49a4e16e68b128803cc2dcea614603632b04eac`, Apache-2.0).

`RAGatouille` is a thin, opinionated Python wrapper that makes
**late-interaction retrieval** (ColBERT v2 / PLAID) usable
without reading the [original ColBERT](https://github.com/stanford-futuredata/ColBERT)
codebase. Where most of the catalog's RAG entries
([`r2r`](../r2r/), [`embedchain`](../embedchain/),
[`paper-qa`](../paper-qa/), [`docsgpt`](../docsgpt/),
[`fast-graphrag`](../fast-graphrag/)) default to single-vector
dense retrieval (one embedding per chunk, cosine similarity),
RAGatouille gives them a one-import escape hatch to
ColBERT-style multi-vector retrieval, where each *token* gets
its own vector and the score is `sum_i max_j cos(q_i, d_j)`.
That difference matters most exactly where dense retrieval is
weakest: out-of-domain corpora, long-tail entity names, and
languages with little embedding-model coverage.

## 1. Install footprint

- `pip install ragatouille` (Python 3.9–3.11; pulls in `colbert-ai`, `faiss-cpu`, `transformers`, `torch`).
- No CLI binary — exposes `from ragatouille import RAGPretrainedModel, RAGTrainer`.
- Index lives on disk under `.ragatouille/colbert/indexes/<name>/` (PLAID-compressed; ~25 % the size of a comparable Faiss `IVF,PQ` index for the same recall).
- Inference runs on CPU for small corpora; GPU recommended above ~10 k documents. No external service required.

## 2. Repo, version, license

- Repo: <https://github.com/AnswerDotAI/RAGatouille>
- Latest release: `0.0.9` (2025-02-11).
- HEAD pinned at this snapshot:
  `e75b8a964a870dea886548f78da1900804749040`.
- License: Apache-2.0. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/AnswerDotAI/RAGatouille/blob/main/LICENSE)
  (sha `f49a4e16e68b128803cc2dcea614603632b04eac`).

## 3. Why it is in the catalog

Most catalog RAG stacks treat retrieval as "embed → cosine →
top-k". RAGatouille is the catalog's reference for the other
branch:

- **Late interaction by default**: `RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")` ships a tested ColBERTv2 checkpoint with a tuned PLAID configuration; no need to assemble training scripts from the academic repo.
- **Domain-adaptation in three lines**: `RAGTrainer.train()` fine-tunes ColBERT on triples mined from the user's own corpus — the practical answer to "our embedding model has never seen our jargon".
- **Drop-in for existing RAG frameworks**: the [`langchain`](../langflow/)-style retriever wrapper makes it a one-line swap inside [`llama-index`](../llama-index/), [`haystack`](../haystack/), or [`dspy`](../dspy/) pipelines — the user keeps their pipeline, only the retrieval call changes.
- **Honest scope**: it does *not* try to be a full RAG framework, a vector DB, or a chunker; it just owns the retrieval step and hands ranked passages back. That makes it composable with [`chonkie`](../chonkie/) (chunking), [`docling`](../docling/) / [`marker`](../marker/) (parsing), and [`qdrant`](../qdrant/) / [`milvus`](../milvus/) / [`lancedb`](../lancedb/) (which it replaces for the retrieval call but can coexist with for hybrid setups).

## 4. Example invocation

```python
from ragatouille import RAGPretrainedModel

# 1. Load the pretrained late-interaction model.
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# 2. Index a small corpus (PLAID-compressed, on disk).
rag.index(
    collection=[
        "ColBERTv2 stores one vector per token and scores via sum-of-max.",
        "PLAID compresses the per-token vectors with residual quantisation.",
        "Late interaction trades a larger index for stronger zero-shot retrieval.",
    ],
    index_name="late-interaction-demo",
)

# 3. Search.
hits = rag.search(query="how does ColBERT score a document?", k=3)
for h in hits:
    print(f"{h['score']:.2f}  {h['content']}")
```

## 5. Where it sits in the catalog

- Retrieval-layer sibling to [`txtai`](../txtai/), [`marqo`](../marqo/), [`weaviate`](../weaviate/), and [`vanna`](../vanna/), but specifically late-interaction rather than dense or hybrid.
- Indexes plug into the framework layer — [`llama-index`](../llama-index/), [`haystack`](../haystack/), [`dspy`](../dspy/), [`r2r`](../r2r/), [`paper-qa`](../paper-qa/) — without rewriting their pipelines.
- Pairs cleanly with [`chonkie`](../chonkie/) for chunking and [`docling`](../docling/) / [`marker`](../marker/) for parsing upstream.
- Complement, not replacement, for [`qdrant`](../qdrant/) / [`milvus`](../milvus/) / [`lancedb`](../lancedb/): hybrid setups keep dense recall in those stores and rerank the top-k with ColBERT scoring.
