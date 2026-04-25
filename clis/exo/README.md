# exo

> Snapshot date: 2026-04. Upstream: <https://github.com/exo-explore/exo>
> Last verified release: **v1.0.71** (2026-04-23, tag SHA `fd707de30b42`).
> License file: [`LICENSE`](https://github.com/exo-explore/exo/blob/main/LICENSE) — Apache-2.0.

A Python CLI / daemon that **clusters heterogeneous everyday devices
into one local LLM inference pool**. Plug a MacBook, a Linux box with
a 3090, and a spare Mac Mini onto the same network and `exo` shards a
single 70B-class model across them, exposing one OpenAI-compatible
endpoint on `http://localhost:52415/v1`.

It earns its catalog slot as a *runtime* in the same family as
`ollama` / `llamafile` / `ramalama`, but with a different axis:
horizontal scale across devices instead of vertical fit on one box.
Discovery is zero-config (UDP broadcast or Tailscale), and the
partitioning strategy is pluggable (`ring memory weighted` is the
default, sharding by available RAM).

## 1. Install

```bash
git clone https://github.com/exo-explore/exo.git
cd exo
pip install -e .          # Python 3.12+, macOS / Linux
# then on every node:
exo
```

The first node started becomes the entrypoint; later nodes auto-join
via UDP broadcast. No central coordinator, no API keys.

## 2. Example

```bash
# On any node in the cluster, after `exo` is running on >=1 device:
curl http://localhost:52415/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"llama-3.1-70b","messages":[{"role":"user","content":"hi"}]}'
```

The model is downloaded once per shard onto the node that will hold
that shard; subsequent `chat/completions` calls reuse the loaded
shards.

## 3. Honest limitation

Network is the bottleneck. On a 1 Gb LAN, inter-node activation
transfer dominates per-token latency; expect single-digit tokens/sec
on a 70B model spread across three commodity boxes. `exo` is the
right tool when you want to *run a model that does not fit on any one
of your machines*, not when you want maximum throughput — for that,
buy more VRAM on a single host and run `llama.cpp` directly.
