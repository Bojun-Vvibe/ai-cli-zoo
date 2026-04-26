# create-llama

> Snapshot date: 2026-04. Upstream: <https://github.com/run-llama/create-llama>

A scaffolder, not an agent. `npx create-llama` interrogates you for a stack
(framework, LLM provider, vector store, data source) and emits a runnable
LlamaIndex-based RAG application — Next.js front-end, FastAPI or Express
back-end, ingestion pipeline, the works. The CLI's output *is* the product;
once it scaffolds, it gets out of the way.

## 1. Install footprint

- `npx create-llama@latest` (no install) or `npm i -g create-llama`.
- Node.js 18+. Output project pulls Python 3.10+ if you pick the FastAPI
  backend.
- No daemon, no persistent state — generates a project directory and exits.

## 2. License

MIT. License file: [`LICENSE.md`](https://github.com/run-llama/create-llama/blob/main/LICENSE.md).

## 3. Latest version

`v0.5.10` (latest tag in <https://github.com/run-llama/create-llama/tags>).

## 4. Models supported

Anything LlamaIndex supports in the picked stack: OpenAI, Anthropic, Gemini,
Mistral, Groq, Ollama, any OpenAI-compatible endpoint. The scaffolder writes
the env-var contract; the generated project is what calls the model.

## 5. MCP support

Not in the scaffolder itself. The generated app can mount MCP servers via the
LlamaIndex MCP client packages, but `create-llama` does not configure that path
out of the box.

## 6. Sub-agent model

None at the CLI layer. The scaffolded app can use LlamaIndex's `AgentWorkflow`
or `ReActAgent` patterns if you pick the agentic template; otherwise the
output is a plain query-engine RAG.

## 7. Telemetry stance

Off. The scaffolder is a one-shot generator with no analytics. Generated apps
inherit whatever telemetry posture you configure for LlamaIndex / your chosen
provider.

## 8. When to choose it

- You want a working RAG project on disk in under five minutes, with the
  framework / vector store / front-end choices already wired together.
- You are starting a LlamaIndex project from scratch and would rather pick from
  a menu than read three quickstart guides.
- You want the canonical "what does a LlamaIndex 0.5-era app look like"
  reference layout to copy patterns from.

## 9. When to skip it

- You already have a project — this is a greenfield generator, not a refactor
  tool.
- You want a runnable agent CLI in the loop (try `aider`, `opencode`,
  `openhands-cli`). `create-llama` writes code; it does not edit it.
- Your stack is not LlamaIndex. Pick a Haystack / LangChain / DSPy scaffolder
  instead.
