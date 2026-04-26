# ag2

> Snapshot date: 2026-04. Upstream: <https://github.com/ag2ai/ag2>

"**AG2: Open-Source AgentOS for AI Agents.**" AG2 (formerly AutoGen,
forked and now stewarded by the ag2ai community) is a Python
multi-agent framework whose core abstraction is the **conversable
agent**: every agent is an LLM endpoint that can send and receive
structured messages from other agents, humans, or tools. AG2 ships a
catalogue of conversation patterns — two-agent chat, group chat,
nested chat, sequential chat, swarm — plus an autonomous code-execution
sandbox, function/tool registration, and human-in-the-loop hooks. The
CLI surface is `pip install ag2` plus the `ag2` console scripts that
ship with extras (`ag2 studio`, etc.); production use is library-mode
in your own Python.

## 1. Install footprint

- `pip install ag2[openai]` (core + OpenAI). Aliased as `pip install
  autogen` for backwards compatibility with the AutoGen 0.2 namespace.
- Per-provider extras: `ag2[anthropic]`, `ag2[gemini]`,
  `ag2[bedrock]`, `ag2[mistral]`, `ag2[cohere]`, `ag2[groq]`,
  `ag2[ollama]`, `ag2[together]`, `ag2[deepseek]`, `ag2[cerebras]`.
- Per-feature extras: `ag2[retrievechat]` (RAG), `ag2[graph-rag-falkor-db]`,
  `ag2[neo4j]`, `ag2[lmm]` (multimodal), `ag2[teachable]`,
  `ag2[browser-use]`, `ag2[crawl4ai]`, `ag2[twilio]`, `ag2[redis]`.
- Python ≥ 3.10, < 3.14.
- Code execution defaults to a local subprocess; opt into Docker
  (`use_docker=True`) or the hosted [E2B](../e2b/) sandbox via
  `ag2[interop-e2b]` for hard isolation.
- Optional companion repo `ag2ai/build-with-ag2` for runnable
  reference apps; `ag2ai/autogen-studio` (separate package) for a
  no-code agent builder GUI.

## 2. Repo + version + license

- Repo: <https://github.com/ag2ai/ag2>
- Latest release: **v0.12.1** (2026-04-25; v1.0 in beta under
  `autogen.beta`)
- License: **Apache-2.0** —
  <https://github.com/ag2ai/ag2/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

OpenAI (Chat + Responses + Azure-OpenAI-compatible), Anthropic
(Claude), Google Gemini (AI Studio + Vertex), AWS Bedrock, Mistral,
Cohere, Groq, DeepSeek, Cerebras, Together, OpenRouter, Ollama (any
local GGUF), LM Studio, vLLM, llama.cpp, any OpenAI-compatible
endpoint via `OAI_CONFIG_LIST` `base_url`. Per-agent model assignment
is the default — declare `llm_config={"model": "gpt-5"}` on the
planner and `llm_config={"model": "ollama/llama-3.3"}` on a cheap
local critic in the same group chat. Multimodal vision via
`ag2[lmm]`.

## 4. MCP support

Yes (client + server). v0.9+ ships first-party MCP integration:
`autogen.mcp.MCPClientSession` mounts any stdio or HTTP MCP server
as tools on a `ConversableAgent`, and the inverse — exposing an AG2
agent as an MCP server — is supported via the `mcp_proxy_server`
helper so AG2 agents become callable from Claude Desktop, IDEs, and
other MCP clients. Sampling and elicitation pass through to the
agent's `human_input_mode`.

## 5. Sub-agent model

Conversable agents all the way down. Patterns:

- **Two-agent chat** — `UserProxyAgent.initiate_chat(assistant, ...)`,
  the canonical AutoGen pattern; the user proxy executes tool calls
  and code, the assistant plans.
- **Group chat** — `GroupChat([alice, bob, carol], speaker_selection_method=...)`
  routes turns by `auto` (LLM picks next speaker), `round_robin`,
  `manual`, or a custom callable.
- **Nested chat** — register a sub-conversation as a tool on a parent
  agent so the parent can dispatch a whole sub-team and receive a
  summarised reply.
- **Swarm** (v0.8+) — handoff-based pattern: each agent declares which
  peers it can transfer to; the swarm runtime threads context
  variables and tool results across handoffs.
- **Sequential chat** — `initiate_chats([{...}, {...}])` runs a
  pipeline of two-agent chats with carry-over.

Memory and code execution are per-agent; group chat shares the
transcript by default (configurable per agent via
`max_consecutive_auto_reply` and message-filter callbacks).

## 6. Telemetry stance

Off by default. AG2 runs in your Python process; egress is your model
providers + your code-execution sandbox. Optional observability via
`ag2[autogen-watsonx]` / `ag2[langfuse]` / `ag2[agentops]` /
OpenTelemetry through standard env vars — all opt-in. The optional
AutoGen Studio GUI runs locally (`autogenstudio ui`) and stores
sessions in a local SQLite DB.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Group chat with LLM-elected speakers, plus a
seven-year-old conversation-pattern cookbook.** AG2's group chat
runtime is the most battle-tested implementation of "let the LLM
decide who speaks next" in the OSS ecosystem — `speaker_selection_method`
is a callable, so you can plug in a router that consults a state
machine, a vector store, or a cheap classifier model. The catalogue
of patterns (sequential, nested, swarm, society-of-mind, reflection,
teachable, captainagent) is genuinely covered by docs and runnable
examples in `build-with-ag2`. Backwards compatibility with the
AutoGen 0.2 import path (`import autogen`) means existing AutoGen
notebooks keep running. Multi-agent code execution with Docker
isolation has been production-grade since 2024.

**Weakness.** **Conversation-shaped, not graph-shaped, and the AG2
vs. AutoGen lineage is confusing.** If your workflow is naturally a
DAG (deterministic stages, conditional edges, parallel branches),
AG2's chat-message metaphor adds overhead — [`langgraph`](../langgraph/)
or [`mastra`](../mastra/) fit better. The AutoGen lineage forked in
2024 into two parallel projects sharing the name; pick AG2 if you
want the conversable-agent line and Apache-2.0 community governance,
and read the v1.0 roadmap doc before pinning. v0.12 is on a deprecation path toward
v1.0 (`autogen.beta` becomes default), so expect API churn over the
next few minor versions.

**When to choose.** You are **building** a Python multi-agent system
where the natural shape is a *conversation* (debate, peer-review,
plan-critique-execute, customer-support escalation, RPG-style
role-play) and you want LLM-elected speaker routing without writing
a graph compiler. Pair with [`mcp-agent`](../mcp-agent/) /
[`composio`](../composio/) for MCP and tool catalogues, with
[`e2b`](../e2b/) for sandboxed code execution, and with
[`agentops`](../agentops/) / [`langfuse`](../langfuse/) for tracing.
Skip if you want a graph DAG (use [`langgraph`](../langgraph/)), a
TypeScript stack (use [`voltagent`](../voltagent/) /
[`mastra`](../mastra/)), a terminal coding agent (use
[`opencode`](../opencode/) / [`claude-code`](../claude-code/) /
[`crush`](../crush/)), or a single-prompt Unix filter (use
[`mods`](../mods/) / [`smartcat`](../smartcat/) / [`llm`](../llm/)).
