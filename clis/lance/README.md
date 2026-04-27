# lance

> Snapshot date: 2026-04. Upstream: <https://github.com/lance-format/lance>
> License file: <https://github.com/lance-format/lance/blob/main/LICENSE>
> Pinned: `v4.0.1` (HEAD `afb5a1fd`). Default branch is `main`.

"**Open lakehouse format for multimodal AI.**" Lance is a columnar
on-disk format (think Parquet, but designed around random access and
vector search) plus a Rust core with Python / Java / JS bindings. You
convert Parquet / Arrow / Pandas / Polars / DuckDB tables to a
`.lance` directory in two lines, then get O(1) random row access,
vector indexes (IVF-PQ, HNSW), full-text search, and zero-copy
versioning over the same files — all without a database server.

## 1. Install footprint

- `pip install pylance` for the Python bindings (ships the Rust
  binary). `cargo add lance` for native Rust use; npm / Java
  artifacts also published.
- No standalone CLI: the operator surface is the `lance` Python /
  Rust API + the higher-level `lancedb` package
  (<https://github.com/lancedb/lancedb>) which adds an embedded
  vector-DB layer on top of the format.
- Storage backends: local filesystem, S3, GCS, Azure Blob, R2 — the
  format is the contract, no daemon required.

## 2. Repo + version + license

- Repo: <https://github.com/lance-format/lance>
- Latest tag at snapshot: **v4.0.1**
- HEAD commit at snapshot: `afb5a1fdc6f119178269206425d21e6ff25bc220`
- License: **Apache-2.0** (`LICENSE`)

## 3. Why it earns a slot

The interesting bit is that Lance is *just a format on disk*. There is
no server to operate, no cluster to size, no separate vector DB to
sync — your training data, your retrieval index, and your evaluation
snapshots all live as the same versioned `.lance` directory you can
`aws s3 sync` or check into a data lake. For multimodal pipelines
(image / audio / video columns alongside embeddings) it sidesteps the
classic "Parquet for storage + Faiss for search + S3 for blobs"
three-system stack with one columnar layout that does all three.
LanceDB is the OLTP-flavored wrapper if you want a queryable embedded
DB; raw `pylance` is the right entry point if you are building your
own training / RAG pipeline and just want better-than-Parquet I/O.
