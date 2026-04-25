# griptape

> Snapshot date: 2026-04. Upstream: <https://github.com/griptape-ai/griptape>

"**Modular Python framework for AI agents and workflows with
chain-of-thought reasoning, tools, and memory.**"
`griptape` is an opinionated agent framework whose core abstraction is
the **Structure** — `Agent` (single LLM loop), `Pipeline` (sequential
tasks), `Workflow` (DAG of parallel tasks). Tasks emit / consume
typed Artifacts (`TextArtifact`, `BlobArtifact`, `CsvRowArtifact`,
`ImageArtifact`); off-prompt data lives in a Task Memory store
(local file / Redis / Postgres / Mongo / OpenSearch) so large tool
outputs do not blow up the context window — the LLM sees a pointer,
the next tool dereferences it.

## 1. Install footprint

- `pip install griptape` (Python ≥ 3.9). Lean core; opt into extras:
  `pip install 'griptape[drivers-prompt-anthropic,drivers-prompt-google,drivers-vector-pinecone,drivers-sql-snowflake,loaders-pdf,loaders-image]'`,
  or `'griptape[all]'` for the full driver matrix.
- Library-first; not a dedicated CLI binary. Wire `Structure`s in a
  Python file, run with `python my_app.py`, or deploy via Griptape
  Cloud (commercial) for managed runtime.
- Auth via per-provider env vars (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  `GOOGLE_API_KEY`, `AWS_*`, …); local models via the OpenAI-compatible
  driver pointed at Ollama / vLLM / LM Studio.

## 2. Repo + version + license

- Repo: <https://github.com/griptape-ai/griptape>
- Latest release: **v1.10.0** (2026-04-20)
- License: **Apache-2.0** —
  <https://github.com/griptape-ai/griptape/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

OpenAI (Chat + Responses), Anthropic, Gemini (AI Studio + Vertex),
Bedrock (Anthropic / Titan / Llama / Mistral / Cohere), Azure OpenAI,
Cohere, Hugging Face Inference, Groq, Perplexity, Ollama, plus any
OpenAI-compatible endpoint (vLLM / LM Studio / llama-cpp / LiteLLM
proxy) via `OpenAiChatPromptDriver(base_url=...)`. Embeddings: OpenAI,
Cohere, HuggingFace, Voyage, Bedrock, Vertex, Ollama. Vector stores:
Pinecone, Marqo, Mongo Atlas, OpenSearch, Pgvector, Qdrant, Redis,
Astra DB, Chroma, Griptape Cloud KB.

## 4. When to use it

- You want **typed Artifacts as the agent's data contract** — an
  `ImageQueryTool` returns a `TextArtifact` describing the image, a
  `SqlTool` returns `CsvRowArtifact`s the next task can iterate; tasks
  do not pass raw strings around and the framework knows what each
  artifact is for.
- You want **off-prompt task memory by default** — a tool that
  returns 50 KB of CSV writes it to the configured store and gives
  the LLM a handle; the next task calls a tool that reads the handle
  back. Stops "tool returned 50 KB → context limit → halt" without
  manual chunking code.
- You want **Workflow as a real DAG** — `task_a >> [task_b, task_c]
  >> task_d` is the wiring; `task_b` and `task_c` run in parallel,
  `task_d` waits on both. Closer in spirit to Airflow than to a
  manager LLM picking the next worker.
- You want a **mature driver matrix** under one Apache-2.0 install
  rather than reaching for a separate provider SDK per driver.

## 5. When NOT to use it

- You want **role-play multi-agent crews** with a manager LLM
  delegating to workers — Griptape's composition is task graphs, not
  agent orgs. Pick [`crewai`](../crewai/) or [`metagpt`](../metagpt/)
  for that.
- You want **typed Pydantic outputs as the headline feature** —
  Griptape uses Artifacts; for `response_model=MySchema` ergonomics
  reach for [`instructor`](../instructor/), [`mirascope`](../mirascope/),
  or [`pydantic-ai`](../pydantic-ai/).
- You want **prompt compilation / few-shot search** — out of scope;
  pick [`dspy`](../dspy/).
- You want a **TUI / interactive chat REPL out of the box** — Griptape
  is a library; ship your own UI on top, or pick [`aichat`](../aichat/)
  / [`shell-gpt`](../shell-gpt/) for the CLI-first idiom.

## 6. Closest alternatives

- [`crewai`](../crewai/) — role-based crews; Griptape is the
  task-DAG alternative without manager-LLM auto-delegation.
- [`agno`](../agno/) — `Agent` / `Team` / `Workflow` primitives plus
  hosted `AgentOS`; richer production surface, similar workflow
  metaphor.
- [`langroid`](../langroid/) — explicit task graphs with typed
  message-passing; closest peer in the "deterministic wiring beats
  manager-LLM" thesis.
- [`pydantic-ai`](../pydantic-ai/) — typed agents via
  `pydantic_graph`; thinner than Griptape on the artifact /
  off-prompt-memory side.
- [`smolagents`](../smolagents/) — code-as-action agents from HF;
  different paradigm (LLM emits Python) vs. Griptape's typed
  task-graph.

## 7. Repo health (snapshot)

- Very active: v1.10.0 on 2026-04-20; release cadence ~3–4 weeks
  through the 1.x line.
- Backed by Griptape AI (commercial parent shipping Griptape Cloud);
  the OSS framework predates the cloud product and remains the
  primary open surface.
- Driver matrix continues to expand each release — new prompt
  drivers, vector stores, and SQL drivers are the dominant changelog
  items.
- Post-1.0 — `Structure` / `Agent` / `Pipeline` / `Workflow` /
  `Artifact` surfaces have been stable since 1.0 (Q3 2024). Driver
  config namespaces moved to `griptape.drivers.<kind>.<provider>` in
  the 1.x line; check release notes when upgrading from 0.x.
