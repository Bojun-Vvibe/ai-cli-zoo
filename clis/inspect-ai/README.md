# inspect-ai

> Snapshot date: 2026-04. Upstream: <https://github.com/UKGovernmentBEIS/inspect_ai>

**LLM evaluation framework from the UK AI Safety Institute (AISI).**
`inspect` is a CLI + Python library for writing, running, and
inspecting LLM evals — the same harness AISI and partners use for
frontier-model safety evaluations. A `Task` declares a `dataset`, a
`solver` chain (prompt → generate → tool-use loop → grade), and a
`scorer`; `inspect eval` runs it concurrently across N samples + M
models with token / latency / cost rollups, and `inspect view` boots
a local web UI with per-sample transcript replay.

## Repo + version + license

- Repo: <https://github.com/UKGovernmentBEIS/inspect_ai>
- Latest version: **`inspect-ai` 0.3.212** (PyPI, 2026-04)
- Latest tag (release branch): `release/2025-11-28` (sha `b547763`)
- HEAD on `main`: `2f110d2` (2026-04-25)
- License: **MIT** —
  <https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/LICENSE>
- License path in repo: `LICENSE`
- License SPDX: `MIT`
- Default branch: `main`
- Language: Python

## Install

```bash
pip install -U inspect-ai
inspect eval my_eval.py --model anthropic/claude-sonnet-4
inspect eval my_eval.py --model openai/gpt-4.1 --limit 50 --epochs 3
inspect view  # transcript browser on :7575
```

## Niche

Production-grade LLM eval harness with **agentic-task support**
(tool use, multi-turn solver chains, browser / shell / Python
sandboxes) and a transcript-replay UI rich enough to actually debug
why a sample failed. Sits next to [deepeval](../deepeval/),
[ragas](../ragas/), [promptfoo](../promptfoo/), [trulens](../trulens/),
but aims at *capabilities + safety* evals (frontier-model risk,
agentic benchmarks like SWE-bench / Cybench / GAIA), not RAG metrics.

## Why it matters

- **The harness behind real frontier-model evaluations** — UK AISI,
  US AISI, Anthropic, OpenAI, METR, and Apollo Research publish
  `inspect`-shaped eval suites; running someone else's published eval
  is `pip install inspect-ai && inspect eval their_task.py` instead of
  porting their bespoke harness.
- **Agentic eval is first-class** — `Solver` chains compose
  `system_message → use_tools(bash, python, web_browser) → generate
  → self_critique → score`; each sample gets its own ephemeral
  sandbox (Docker / k8s) so the model can `apt install` and `git
  clone` without leaking across samples. Closest catalog peer to
  [swe-agent](../swe-agent/) at the eval layer.
- **Transcript UI is the killer feature** — `inspect view` renders
  every sample as a tree (messages, tool calls, tool outputs, scorer
  reasoning) with diff against expected, side-by-side comparison
  across models, and replay on a single sample with edited inputs;
  the eval debugger this catalog otherwise lacks.
- **Provider matrix** — OpenAI, Anthropic, Google (AI Studio + Vertex),
  Bedrock, Together, Groq, Mistral, Cohere, Ollama, vLLM, LiteLLM,
  any OpenAI-compatible URL, plus a `mockllm` for deterministic CI.
- Funded by a public-sector AI safety institute, MIT-licensed, no
  hosted-control-plane requirement; the trust profile most eval
  harnesses can't match.
