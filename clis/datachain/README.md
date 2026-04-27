# datachain

- **Repo:** https://github.com/datachain-ai/datachain
- **Version:** 0.54.0 (released 2026-04-25; HEAD pinned 7b23faf)
- **License:** Apache-2.0 (`LICENSE`, blob SHA `eea7a3bf5d8b5257b02835ed501b91ebf07eb3bf`)
- **Category:** Multimodal data wrangling / dataset versioning for AI

## What it does

A Python library + `datachain` CLI for building, versioning, and querying
**unstructured-data datasets** (images, video, audio, PDFs, text) the way
pandas / polars handle tabular data. Backed by a SQLite (or Postgres /
ClickHouse) catalog that stores per-file metadata, embeddings, and LLM
annotations as columns; the dataset itself stays in object storage
(S3 / GCS / Azure / local) and is referenced by content-addressed
manifest, so a 10 TB image corpus enriched with CLIP embeddings + GPT-4o
captions versions in seconds without copying bytes.

## Install

```sh
pip install 'datachain[examples]'
datachain query my_dataset.py        # run a chain script
datachain ls s3://bucket/path        # browse object storage
datachain dataset list               # show versioned datasets
```

## Why it's interesting

The data-engineering peer to [`dvc`](https://dvc.org) for the
unstructured-data + LLM-annotation era. Where peers like
[`fiftyone`](../fiftyone/) target visual-AI exploration with a UI
in the loop, datachain stays headless and SQL-shaped: a `Chain` is a
lazy DAG of `gen` (1→N), `map` (1→1), `filter`, `agg`, and
`save` operators that compile to a single query plan, so adding a
"caption every image with GPT-4o, then keep only the captions
mentioning a dog" stage is a 10-line script that fans out across
a thread pool, checkpoints to the catalog, and survives an OOM
mid-run by resuming from the last persisted version. UDFs declare
their input columns with type hints; Pydantic models become row
schemas; the result is one diffable Python file per dataset version
that a coding-agent CLI can author and re-run.
