# potpie

- **Repo:** https://github.com/potpie-ai/potpie
- **Version:** `1.0.2` (latest release, 2026-04-22)
- **License:** Apache-2.0 (`LICENSE`)
- **Language:** Python
- **Install:** `git clone https://github.com/potpie-ai/potpie && cd
  potpie && ./start.sh` (Docker Compose), or use the hosted
  version at `app.potpie.ai`. A `potpie` CLI ships in the same repo
  for indexing repos and invoking custom agents from the terminal.

## One-line summary

A **codebase-knowledge-graph + spec-driven-development** platform:
potpie ingests a repo, builds a graph of files / functions / classes
/ call-edges using tree-sitter + an LSP layer, indexes it into Neo4j
+ a vector store, then exposes a library of **codebase-aware agents**
(debugging, code-review, integration-test-writer, low-level-design,
codebase-Q&A, custom) that ground every answer in a graph traversal
across the indexed repo rather than a flat RAG search.

## What it does

The pitch is: **agents that actually understand your codebase
structure, not just blob-search it.** The system has three layers:

- **Parsing + indexing** (`/parsing` service): on `potpie parse
  <repo>`, tree-sitter parses every supported file (Python, TS/JS,
  Go, Java, C#, Rust, Ruby, Kotlin) into a per-language AST, then a
  symbol resolver builds a **knowledge graph** in Neo4j with nodes
  for files / classes / functions / methods and edges for `CALLS`,
  `INHERITS`, `IMPORTS`, `IMPLEMENTS`, `CONTAINS`. Function bodies
  are also embedded into a vector store keyed back to graph node
  IDs, so every embedding hit comes with structural context.
- **Pre-built agents**: `debugging_agent` (takes a stack trace,
  walks the call graph from each frame), `codebase_qna_agent`
  (Q&A grounded in graph + vector hits), `unit_test_agent`
  (generates pytest / jest from a target function plus its
  dependencies), `integration_test_agent`, `low_level_design_agent`
  (proposes class/method shapes for a feature description),
  `code_changes_agent` (analyzes a diff for impact via the call
  graph), `code_generation_agent`. Each agent is a LangGraph state
  machine with explicit tools for "fetch node by name", "expand
  one hop in graph", "vector search within subtree".
- **Custom agent builder**: a YAML / web-form surface where you
  declare an agent as `name + system_prompt + tools (subset of
  graph + vector tools) + termination condition`, and it's
  immediately invokable via the same API as the pre-built ones.
  This is the spec-driven-development loop — you write a spec, the
  custom agent grounds it in your real codebase.
- **Multi-LLM**: works with OpenAI, Anthropic, Gemini, AWS Bedrock,
  Ollama; provider is per-agent, so you can route the cheap
  Q&A agent to a small local model and the design agent to a
  frontier model.

```bash
git clone https://github.com/potpie-ai/potpie && cd potpie
./start.sh                              # docker-compose up: API, neo4j, postgres, qdrant
potpie parse https://github.com/me/my-repo
potpie agent run debugging_agent \
  --repo my-repo --branch main \
  --prompt "Why does fetch_user fail when org_id is null?"
```

## When to choose it

- You're working in a **large, established codebase** where flat
  RAG ("embed every file, top-k") gives bad answers because the
  real signal is "what calls what" — the graph layer is the
  point.
- You want **multiple specialized codebase agents** sharing one
  index (debugging, test-gen, design, review) instead of running a
  separate tool per task with its own embedding pass.
- You want **self-hostable** with first-class support for local
  models (Ollama) and your own Neo4j / Qdrant — the docker-compose
  is the canonical install.

## When NOT to choose it

- You want a **lightweight in-editor agent** for a small repo —
  potpie's parsing + Neo4j + vector store overhead doesn't pay off
  on small codebases. Use a CLI like `aider` / `claude-code` /
  `opencode` instead.
- You don't have Docker / can't run Neo4j — there's no embedded /
  in-memory mode; the graph store is required.
- Your codebase is in a language not yet supported by the parser
  (today: Python, TS/JS, Go, Java, C#, Rust, Ruby, Kotlin) — you'd
  fall back to vector-only, which negates the unique value.
