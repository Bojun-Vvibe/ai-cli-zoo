# langroid

> Snapshot date: 2026-04. Upstream: <https://github.com/langroid/langroid>

"**Harness LLMs with Multi-Agent Programming.**"
`langroid` is a Python framework (CMU / UW-Madison alumni-built) that
treats LLM agents as message-passing actors with typed `ChatDocument`
mailboxes. The headline pattern is "task graph": each `Task` wraps an
`Agent`, owns its own conversation history, and forwards messages to
sub-tasks via deterministic `addSubTask` wiring rather than a manager
LLM auto-spawning workers. The whole thing runs without LangChain /
LlamaIndex underneath — Langroid implements its own retrieval,
tool-calling, and orchestration primitives.

## 1. Install footprint

- `pip install langroid` (Python ≥ 3.11). Slim core; opt into extras
  for vector stores and providers: `pip install
  'langroid[postgres,mysql,hf-embeddings,unstructured,chromadb,qdrant,lancedb,meilisearch,neo4j,litellm,docling,mcp]'`,
  or `'langroid[all]'`.
- Library-first; not a dedicated CLI binary. Wire agents in a
  Python file and run with `python my_app.py`. The repo ships ~80
  example scripts under `examples/` covering RAG, multi-agent debate,
  SQL agents, and DocChatAgent.
- Auth via per-provider env vars (`OPENAI_API_KEY`, `GEMINI_API_KEY`,
  `ANTHROPIC_API_KEY`, …); local models via `OPENAI_API_BASE` pointing
  at Ollama / vLLM / LM Studio / llama-cpp-python.

## 2. Repo + version + license

- Repo: <https://github.com/langroid/langroid>
- Latest release: **0.61.1** (2026-03-25)
- License: **MIT** —
  <https://github.com/langroid/langroid/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

OpenAI (Chat + Responses), Anthropic, Gemini (AI Studio + Vertex),
Groq, Cerebras, DeepSeek, OpenRouter, Together, Fireworks, plus any
LiteLLM-routed provider via the bundled `[litellm]` extra. Local
inference via the OpenAI-compatible URL pattern: Ollama, vLLM,
LM Studio, llama-cpp-python, mlx-lm — every one is reached by setting
`OpenAIGPTConfig(api_base="http://localhost:11434/v1", chat_model="ollama/llama3.1")`.
Embeddings: OpenAI, Sentence-Transformers, FastEmbed, Cohere, Gemini.
Vector stores: Qdrant (default), Chroma, LanceDB, Postgres + pgvector,
Meilisearch, Weaviate, Pinecone.

## 4. When to use it

- You want **deterministic multi-agent wiring** — agent A always
  forwards to agent B, escalates to C only on a typed failure — rather
  than handing a manager LLM the keys and hoping it picks the right
  worker. `Task` graphs are explicit Python; no manager prompt to
  babysit.
- You are building a **typed-message RAG agent** where retrieval,
  citation, and answer composition are separate `Agent` subclasses
  (`DocChatAgent`, `SQLChatAgent`, `TableChatAgent`, `LanceRAGTaskCreator`)
  rather than one mega-prompt.
- You want a framework that **does not pull LangChain / LlamaIndex**
  as a transitive dep and still gives you tool-calling, function
  registration, vector-store retrieval, and document chunking.
- You need **MCP client support** — the `[mcp]` extra mounts MCP
  servers as callable tools on any `ChatAgent`.

## 5. When NOT to use it

- You want **a hosted UI / playground** — Langroid is a Python
  library; ship your own FastAPI/Streamlit/Chainlit on top. Reach
  for [`agno`](../agno/)'s AgentOS or [`crewai`](../crewai/)'s
  hosted runtime when you want batteries-included serving.
- You want **role-play multi-agent crews** ("CEO directs CTO directs
  engineer") with auto-delegation — Langroid favours explicit task
  wiring; [`crewai`](../crewai/) and [`metagpt`](../metagpt/) are the
  role-play idiom.
- You want **prompt compilation / optimisation** — out of scope; pick
  [`dspy`](../dspy/) for that.
- You want **TypeScript / Go / Rust** bindings — Python only.

## 6. Closest alternatives

- [`crewai`](../crewai/) — role-based crews with a manager-LLM
  process; Langroid is the explicit-graph alternative.
- [`agno`](../agno/) — `Agent` + `Team` + `Workflow` primitives plus
  a hosted `AgentOS` runtime; richer production surface than Langroid.
- [`pydantic-ai`](../pydantic-ai/) — Pydantic-typed agent loop,
  graph composition via `pydantic_graph`; closest peer in the
  "typed Python, no DSL" niche.
- [`smolagents`](../smolagents/) — code-as-action agents from HF;
  thinner than Langroid, no built-in RAG stack.
- [`metagpt`](../metagpt/) — software-team role play; a different
  thesis (auto-spawn org chart) from Langroid's explicit wiring.

## 7. Repo health (snapshot)

- Active development: 0.61.x line landed through Q1 2026; release
  cadence ~2 weeks.
- ~80 runnable examples under `examples/` cover the documented
  patterns end-to-end (RAG, multi-agent debate, SQL agent, DocChat,
  Lance + RAG, MCP client).
- Pre-1.0 — `Task` / `ChatAgent` / `DocChatAgent` surfaces have been
  stable across the 0.5x → 0.6x line, but check the changelog before
  pinning.
