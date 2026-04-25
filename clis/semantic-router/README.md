# semantic-router

> Snapshot date: 2026-04. Upstream: <https://github.com/aurelio-labs/semantic-router>

**Sub-millisecond intent routing via embedding cosine similarity, not
an LLM call.** `semantic-router` defines `Route` objects ("book a
flight", "refuse: politics", "tool: search_docs") each backed by 5–20
example utterances; at request time it embeds the input once and
picks the nearest route by cosine — typically 1–10 ms vs the
500–2000 ms of an LLM-classifier hop. Used as the front door of an
agent stack to short-circuit guardrails, tool selection, and model
tier routing without paying for a planner call.

## Repo + version + license

- Repo: <https://github.com/aurelio-labs/semantic-router>
- Latest release: **`v0.1.12`** (2025-11-18)
- HEAD on `main`: `371cbf7`
- License: **MIT** —
  <https://github.com/aurelio-labs/semantic-router/blob/main/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `main`
- Language: Python

## Install

```bash
pip install -U "semantic-router[local]"   # add fastembed / hf locally
# minimal end-to-end
python - <<'PY'
from semantic_router import Route, SemanticRouter
from semantic_router.encoders import HuggingFaceEncoder
politics = Route(name="politics", utterances=["who should I vote for", "is the president doing a good job"])
chitchat = Route(name="chitchat", utterances=["how are you", "what's up"])
rl = SemanticRouter(encoder=HuggingFaceEncoder(), routes=[politics, chitchat], auto_sync="local")
print(rl("are you free for lunch?").name)   # -> chitchat
PY
```

## Niche

The "**deterministic, cacheable router that runs before the LLM**"
slot. Where [guardrails](../guardrails/) validates *outputs* and
[instructor](../instructor/) shapes *outputs*, semantic-router shapes
the *dispatch* — which prompt template, which tool subset, which
model tier, which refusal — using only an embedding lookup. Pairs
naturally with [outlines](../outlines/) (structured generation
once a route is chosen) and any vector store ([qdrant](../qdrant/),
[chroma](../chroma/), [lancedb](../lancedb/)) as the index backend.

## Why it matters

- 1–10 ms p50 routing decisions on a single embedded vector — orders
  of magnitude cheaper than an LLM-classifier hop, and the only path
  for sub-100ms voice-agent turn-taking budgets.
- `auto_sync` keeps the route index consistent across local /
  Pinecone / Qdrant / Postgres backends, so the same route table can
  back a laptop notebook and a fleet of replicas.
- First-class `dynamic` routes attach a function-call schema to a
  route, turning the router into a *function dispatcher* you can
  plug in front of OpenAI / Anthropic tool calls and skip the
  planner entirely for known-shape requests.
- 0.1.x line is the actively maintained rewrite — encoder layer now
  supports HuggingFace, OpenAI, Cohere, FastEmbed, Jina, VoyageAI,
  Bedrock, Vertex out of the box.
