# lmcache

> Snapshot date: 2026-04. Upstream: <https://github.com/LMCache/LMCache>

**Cross-engine, cross-instance KV-cache layer that lets vLLM / SGLang
share precomputed attention state instead of recomputing it on every
request.** LMCache is a Python+C++/CUDA library plus an out-of-process
storage backend (CPU RAM, local disk, Mooncake / Redis / NIXL distributed
stores) that intercepts an inference engine's KV-cache reads/writes and
serves cache hits for *any* shared token prefix — not just the head of a
prompt, but arbitrary mid-prompt blocks (RAG chunks, few-shot examples,
agent tool histories) that recur across requests but live at variable
positions. The result is "first-token latency on cache hit ~ retrieval
time, not prefill time" for workloads where the same chunks are
re-injected by RAG pipelines with high frequency.

## Repo + version + license

- Repo: <https://github.com/LMCache/LMCache>
- Latest release: **`v0.4.4`** (2026-04-22)
- HEAD on `dev`: `47289ba`
- License: **Apache-2.0** —
  <https://github.com/LMCache/LMCache/blob/dev/LICENSE>
- License path in repo: `LICENSE` (11357 bytes, full Apache-2.0 text)
- Default branch: `dev`
- Language: Python (+ C++/CUDA kernels for cache offload + page management)

## Install

```bash
pip install lmcache                                  # pulls the matching torch + cuda wheels
# Drop-in vLLM integration: launch vLLM with the LMCache connector
LMCACHE_CONFIG_FILE=./lmcache.yaml \
  vllm serve meta-llama/Llama-3.1-8B-Instruct \
    --kv-transfer-config '{"kv_connector":"LMCacheConnectorV1","kv_role":"kv_both"}'
# point any OpenAI client at http://127.0.0.1:8000/v1
```

```yaml
# lmcache.yaml — typical "RAM tier + local SSD tier" config
chunk_size: 256
local_cpu: True
max_local_cpu_size: 32       # GiB of host RAM for KV pages
local_disk: "file:///mnt/nvme/lmcache"
max_local_disk_size: 512     # GiB on the SSD tier
remote_url: null             # set to redis:// or mooncake:// for cross-instance sharing
remote_serde: "naive"
```

## Niche

The "**KV cache as a separately-tiered, cross-engine, cross-instance
substrate**" slot. Engines like [`vllm`](../vllm/) and
[`sglang`](../sglang/) ship their own prefix-cache (PagedAttention,
RadixAttention) but those caches are per-process, RAM-bound, and only
work on the *prefix* of a request. LMCache is positional-arbitrary
(`CacheBlend`-style "stitch precomputed KV chunks at any position"),
multi-tier (HBM ↔ host RAM ↔ NVMe ↔ remote Redis/Mooncake), and
cross-instance (a request that lands on replica B can pull cache that
was warmed by replica A). The win is biggest on agentic / RAG
workloads where the same retrieved passages or tool transcripts get
re-injected across hundreds of independent requests.

## Why it matters

- **Position-independent KV reuse** — `CacheBlend` lets you cache the
  KV for a RAG chunk *once* and stitch it into any future request that
  contains that chunk, regardless of where in the prompt it lands;
  reported 3-10× TTFT reduction on multi-turn QA + RAG benchmarks vs.
  vanilla prefix-cache.
- **Multi-tier storage with explicit eviction** — host RAM tier for
  microsecond hits, local NVMe tier for tens-of-µs warm cache, optional
  Mooncake / Redis / NIXL remote tier for fleet-wide sharing; per-tier
  capacity caps and an LRU/LFU policy you configure in YAML rather than
  trusting the engine's opaque default.
- **First-class production deployments** — Helm chart + Kubernetes
  operator (`vllm-production-stack`) ship in adjacent LMCache-org
  repos; the canonical reference is "deploy vLLM with the LMCache
  connector + a Mooncake remote tier and watch p50 TTFT drop on the
  RAG endpoint".
- **Prefill / decode disaggregation** — LMCache is the KV-transfer
  layer in disaggregated-prefill setups where prefill GPUs and decode
  GPUs are physically separate; the connector pushes the prefill KV
  over RDMA / NIXL to the decode pool instead of recomputing on the
  decode side.
- **Apache-2.0**, integration is connector-shaped (no fork of vLLM
  required), and the project ships day-zero support for new vLLM
  releases (the `kv_connector` interface is the contract).
