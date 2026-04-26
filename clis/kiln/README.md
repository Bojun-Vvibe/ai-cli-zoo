# kiln

> Snapshot date: 2026-04. Upstream: <https://github.com/Kiln-AI/Kiln>
> License files: <https://github.com/Kiln-AI/Kiln/blob/main/LICENSE.txt>
> (top-level summary), <https://github.com/Kiln-AI/Kiln/blob/main/libs/core/LICENSE.txt>
> (MIT, Python core), <https://github.com/Kiln-AI/Kiln/blob/main/libs/server/LICENSE.txt>
> (MIT, FastAPI server), and `app/EULA.md` (Kiln Desktop EULA, free-to-use
> closed binary).
> Pinned: `v0.28.0` (2026-04-22).

A **dataset-, eval-, and fine-tune-centric agent platform** from
Chesterfield Laboratories. Where most catalog entries are either
"library you `import`" ([`pydantic-ai`](../pydantic-ai/),
[`langgraph`](../langgraph/), [`crewai`](../crewai/)) or "terminal
chat that edits files" ([`aider`](../aider/),
[`opencode`](../opencode/)), Kiln is a **desktop app + Python SDK +
HTTP server triplet** built around the artifact most teams are
actually missing: a versioned, schema-typed **dataset of task
input → expected output → human ratings → model traces** that you
can drive evals, RAG retrieval-grading, and SFT / DPO fine-tunes from.

## 1. Install footprint

- **Desktop app (recommended for hands-on labeling)**: `.dmg` /
  `.exe` / `AppImage` from the GitHub Releases page. Free under the
  Kiln AI Desktop EULA (closed binary, no source). The desktop is
  the rating + dataset-curation surface; under the hood it embeds
  the same `kiln-server` FastAPI process the SDK talks to.
- **Python SDK (open-source path)**: `pip install kiln-ai` — pulls
  Pydantic v2, `httpx`, `tiktoken`, `litellm` (for the 100-provider
  routing layer). Python ≥ 3.10. MIT-licensed (see `libs/core`).
- **Server**: `pip install kiln-server` then `python -m
  kiln_server.server_main` — boots a FastAPI app on `:8757` that
  the desktop and SDK both speak to. MIT-licensed (`libs/server`).
- **CLI surface**: no top-level `kiln` binary like `aider`; the
  Python entry points are `python -m kiln_server.server_main`,
  `python -m kiln_ai.adapters.fine_tune.dpo` etc. Day-to-day driver
  is the desktop UI or the SDK from a notebook.

## 2. Repo + version + license

- Repo: <https://github.com/Kiln-AI/Kiln>
- Latest release: **v0.28.0** (2026-04-22)
- License: **MIT** for the Python libraries
  (`libs/core/LICENSE.txt`, `libs/server/LICENSE.txt`); Kiln AI
  Desktop EULA for the `app/` Electron binary (free download, source
  not redistributable). Top-level explainer at `LICENSE.txt`.
- Default branch: `main`
- Language: Python (libraries + server) + TypeScript / Svelte
  (Electron desktop)

## 3. Models supported

LLM routing is via [`litellm`](../litellm/), so Kiln inherits the
~100-provider matrix: OpenAI (GPT-4o / 4.1 / o-series), Anthropic
(Claude 3.5 / 3.7 / 4 family), Google (Gemini 1.5 / 2.x via AI
Studio + Vertex), AWS Bedrock, Azure OpenAI, Mistral, Cohere, Groq,
DeepSeek, xAI Grok, Together, Fireworks, OpenRouter, Perplexity,
Ollama (local), llama.cpp, vLLM, any OpenAI-compatible. **Fine-tune
backends** are first-class and are the differentiator: OpenAI
fine-tuning API, Together AI, Fireworks, Vertex AI tuning, Unsloth
(local LoRA / QLoRA), and `kiln_ai.adapters.fine_tune.dpo` for
direct DPO from rated dataset rows. Embeddings + rerankers route
through the same `litellm` layer.

## 4. MCP support

Yes (client). The agent runner can mount third-party MCP servers as
tool sources in a task definition; tool selection is captured in the
trace alongside model output, which is what makes the dataset row
re-trainable later. No first-party `kiln --mcp-server` mode — the
server is HTTP / FastAPI, not stdio MCP, since the desktop is the
intended client.

## 5. Sub-agent model

Kiln's unit is a **Task** (typed `input_json_schema` →
`output_json_schema` with a free-text `instruction`), not an Agent.
Tasks compose via:

- **Multi-step Tasks** — a Task can call sub-Tasks. Each sub-Task
  has its own model + prompt + dataset, so the trace records every
  hop and ratings can be attached at any level.
- **Eval Tasks** — Tasks whose output is a graded score on another
  Task's output. Used as LLM-as-judge graders that themselves get
  rated by humans, so the grader can be fine-tuned.
- **Synthetic-data Tasks** — Tasks that generate input rows for
  another Task; the curated outputs become training data.

Compared with [`crewai`](../crewai/) / [`agency-swarm`](../agency-swarm/),
the orientation is **dataset-first, not agent-first**: the loop you
optimize is "label → eval → fine-tune → eval again", and the agent
graph is bookkeeping in service of that.

## 6. Telemetry stance

Off by default. Self-hosted FastAPI server stores Tasks + Runs +
Ratings in a local SQLite by default (Postgres optional). Egress is
your configured LLM / fine-tune providers and nothing else. The
desktop app is local-only — there is no Kiln cloud control plane.
Optional opt-in OpenTelemetry export via standard OTel env vars.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **A real dataset is the artifact, and fine-tunes
are a one-click operation off it.** Most catalog entries treat
dataset curation and fine-tuning as someone else's problem — Kiln
treats the **rated dataset row** (`task_input`, `task_output`,
`source` ∈ `{human, synthetic, model}`, `repaired_output`,
`rating: 1-5 + reasoning`, `trace_id`) as the system of record.
From the desktop you label runs, then `Fine-tune` opens a wizard
that targets OpenAI / Together / Fireworks / Vertex / Unsloth (LoRA
on your own GPU) with the dataset auto-formatted, the train/val
split done, and the resulting model registered as a new provider in
the same Task — so the next run uses the tuned model and the next
ratings flow back into the same dataset for v2. DPO is symmetric:
two rated outputs become a `(chosen, rejected)` pair without
re-labeling. LLM-as-judge graders are themselves Tasks, so a bad
grader can be fine-tuned the same way.

**Weakness.** **Not a coding agent and not a CLI-shaped tool.**
The day-to-day surface is the Electron desktop or a Jupyter
notebook calling the Python SDK; there is no `kiln "fix this bug"`
terminal mode like [`aider`](../aider/) /
[`opencode`](../opencode/) / [`crush`](../crush/). The desktop
binary is closed-source (free EULA, but not redistributable) — fully
air-gapped users who refuse closed binaries are stuck running the
MIT server + SDK and building their own UI. The fine-tune wizard
covers the major providers but local full-parameter SFT / RLHF
pipelines (DeepSpeed / FSDP / Megatron) are explicitly out of scope
— Kiln punts to Unsloth for the on-GPU path, which caps at
LoRA / QLoRA on consumer hardware.

**When to choose.** You are **building a domain-specific model**
(internal classifier, support-ticket router, structured-extraction
pipeline) and the bottleneck is *data quality + iteration speed*,
not "which framework do I import". You want a **typed dataset that
survives the framework you happen to be using this quarter**, with
human ratings, synthetic augmentation, multi-stage evals, and a
fine-tune button that closes the loop. Pair with
[`pydantic-ai`](../pydantic-ai/) when you want to ship the trained
Task as a typed Python service, with [`langfuse`](../langfuse/) /
[`helicone`](../helicone/) when you also need request-level
production observability across the fleet, or with
[`promptfoo`](../promptfoo/) when the eval shape is "matrix of
prompts × models × cases" rather than "rated dataset of runs".
Skip if your use case is "edit my repo with natural language" (use
[`aider`](../aider/) / [`opencode`](../opencode/) /
[`continue`](../continue/) instead) or if your team will not adopt
a desktop labeling app.
