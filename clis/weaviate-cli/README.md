# weaviate-cli

> Snapshot date: 2026-04. Upstream: <https://github.com/weaviate/weaviate-cli>
> License file: <https://github.com/weaviate/weaviate-cli/blob/main/LICENSE>
> Pinned: `v3.4.0` (HEAD `1a469a0c`). Default branch is `main`.

"**Command-line interface for Weaviate.**" `weaviate-cli` is the
official Python-packaged CLI for operating a Weaviate vector database
instance — local, self-hosted, or Weaviate Cloud — without writing
client code. Where the existing [`weaviate`](../weaviate/) entry in
this catalog covers the *server*, this entry covers the *operator
surface*: schema CRUD, bulk import, backup / restore, tenant
management, and ad-hoc query from a shell.

## 1. Install footprint

- `pip install weaviate-cli` (Python 3.9+). Single binary entry
  point: `weaviate`.
- Talks to any Weaviate endpoint over HTTPS — set `WCD_URL` /
  `WCD_API_KEY` env vars (or `--url` / `--api-key` flags) and you
  drive a managed cluster from the same shell that drives a `docker
  compose` dev box on `:8080`.
- Subcommands map 1:1 to the resource model: `weaviate collection
  create | list | delete`, `weaviate tenant create`, `weaviate data
  import`, `weaviate query near-text`, `weaviate backup create`.

## 2. Repo + version + license

- Repo: <https://github.com/weaviate/weaviate-cli>
- Latest tag at snapshot: **v3.4.0**
- HEAD commit at snapshot: `1a469a0c...`
- License: **BSD-3-Clause** (`LICENSE`)

## 3. Why it earns a slot

The Weaviate Python / TS clients are great when you are writing
application code, but they are wrong tool for the day-to-day operator
loop — "create a collection, push a JSON Lines dump, snapshot it,
swap it into prod, fail back if recall drops." `weaviate-cli` is the
shell-native equivalent: every operation is one command with stable
exit codes, `--output json` everywhere, and the same flags work
against a local container and a managed cluster, which makes it
trivially CI-able. Pair it with [`weaviate`](../weaviate/) (the
server) and one of the embedding-producing CLIs in this catalog to
get a complete shell-only RAG ingest pipeline.
