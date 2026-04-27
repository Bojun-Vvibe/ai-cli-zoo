# vespa

> Snapshot date: 2026-04. Upstream: <https://github.com/vespa-engine/vespa>
> License file: <https://github.com/vespa-engine/vespa/blob/master/LICENSE>
> Pinned: `v8.677.31` (HEAD `d40d356f`). Default branch is `master`.

"**The open big-data serving engine.**" Vespa is a distributed
serving platform that combines structured search, vector search
(HNSW), full-text retrieval (BM25 / nativeRank), and per-query ML
inference (ONNX / XGBoost / TensorFlow) inside a single ranking
expression evaluated at query time. Originally Yahoo's production
search backbone, open-sourced in 2017; it is the rare engine that
treats "rank with a learned model over the candidate set" as a
first-class server-side operation rather than a re-rank step bolted
on top of a vector DB.

## 1. Install footprint

- `vespa` CLI: `brew install vespa-cli` (Mac) or grab a release
  binary. Drives a local Docker single-node, Vespa Cloud, or
  self-hosted clusters with the same subcommands (`vespa deploy`,
  `vespa feed`, `vespa query`, `vespa status`).
- Server: `docker run -p 8080:8080 vespaengine/vespa` for a
  single-node dev box; production deployments are JVM-based clusters
  with separate config / content / container nodes.
- Application package is a directory of XML + schema files +
  optional ONNX model artifacts; `vespa deploy <dir>` ships it.

## 2. Repo + version + license

- Repo: <https://github.com/vespa-engine/vespa>
- Latest tag at snapshot: **v8.677.31**
- HEAD commit at snapshot: `d40d356f...`
- License: **Apache-2.0** (`LICENSE`)

## 3. Why it earns a slot

Most "vector DB" entries in this catalog are good at one axis:
similarity search over embeddings. Vespa's pitch is that real
production retrieval almost never wants pure ANN — it wants
"hybrid BM25 + vector + structured filters + a learned
re-ranker, evaluated in one round-trip with strict latency SLOs." Its
ranking expressions let you call an ONNX cross-encoder over the top-K
candidates *server-side*, so the embedding model, the retrieval
index, and the re-ranker live in one process with one tail latency
budget. Heavyweight to operate (real cluster, real JVM tuning, real
schema discipline) — pick it when your workload is search /
recommendation at scale, not when you need a 50-line RAG demo.
