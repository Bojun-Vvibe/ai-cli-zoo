# jina

- **Repo:** https://github.com/jina-ai/serve
- **Version:** v3.28.0 (latest release; HEAD pinned 0f32b2a)
- **License:** Apache-2.0 (`LICENSE`)
- **Category:** Multimodal AI serving framework

## What it does

A cloud-native framework for building and serving multimodal AI services
as composable pipelines (`Flow`s) of `Executor`s. Each Executor wraps a
model or processing step and is exposed over gRPC, HTTP, and WebSockets
with automatic batching, replication, and sharding; the `jina` CLI
scaffolds, runs, and deploys these flows locally, in Docker, or on
Kubernetes.

## Install

```sh
pip install jina
jina new my-service          # scaffold an Executor + Flow
jina flow --uses flow.yml    # serve locally
```

## Why it's interesting

Predates the current agent-framework wave by years and is still one of
the most production-tested ways to wrap an embedding model, reranker, or
multimodal pipeline behind a real RPC surface — including streaming,
health checks, OTel tracing, and Prometheus metrics out of the box. The
YAML-defined Flow abstraction lets you compose CLIP, Whisper, a vector
DB, and a custom Executor into one deployable unit without writing
gRPC servers or Dockerfiles by hand, which is a niche the LangChain /
LlamaIndex serving stories never fully covered.
