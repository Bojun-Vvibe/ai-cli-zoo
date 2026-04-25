# mastra

> Snapshot date: 2026-04. Upstream: <https://github.com/mastra-ai/mastra>

**TypeScript-native agent framework with a real CLI and dev surface.**
Mastra is what Next.js looks like for agents: declarative `Agent`,
`Tool`, `Workflow`, and `Memory` primitives in TypeScript with
end-to-end types from input to tool call to output, plus a `mastra`
CLI that scaffolds projects, runs a local dev playground (HTTP +
chat UI + workflow visualizer), wires evals, and packages the agent
as a deployable Hono / Cloudflare Workers / Vercel / Node service.

## Repo + version + license

- Repo: <https://github.com/mastra-ai/mastra>
- Latest release: **`@mastra/core` 1.27.0** (2026-04-24)
- HEAD on `main`: `b97a059`
- License: **Apache-2.0** (with `ee/` directory carved out under a
  separate EE license) —
  <https://github.com/mastra-ai/mastra/blob/main/LICENSE.md>
- License path in repo: `LICENSE.md`
- License SPDX: `Apache-2.0`
- Default branch: `main`
- Language: TypeScript

## Install

```bash
npm create mastra@latest    # interactive scaffold
npx mastra dev              # playground UI + HTTP server
npx mastra build            # compile to deployable bundle
npx mastra deploy           # to configured target
```

## Niche

The TypeScript answer to [agno](../agno/) / [crewai](../crewai/) /
[langgraph](../langgraph/). Targets full-stack TS teams who want a
typed agent stack without dropping into Python, with every primitive
(Agent, Tool, Workflow, Memory, RAG, Eval) shipped from one
`@mastra/*` family and one CLI.

## Why it matters

- **`mastra dev` is the killer demo** — boots a chat playground, an
  OpenAPI surface for every agent + workflow, a Workflow-as-DAG
  visualiser with per-step input/output inspection, and a tool-call
  trace view, all on `localhost:4111` with hot reload on file change.
  Closer to a Next.js dev experience than any Python-first framework
  manages.
- **End-to-end types**: `Agent<Input, Output>` flows through
  `tools: z.object({...})` Zod schemas into `workflow.step({...})`
  into the HTTP handler the CLI generates — `tsc` catches "I changed
  the tool's return shape and forgot to update the agent" before it
  ships.
- **Provider-mixing is one line per agent** — `model: openai("gpt-4.1")`,
  `model: anthropic("claude-sonnet-4")`, `model: ollama("qwen3-coder")`,
  picked per agent or per workflow step via the AI SDK provider matrix
  (~20 providers).
- **Deployable from the same repo** — `mastra build` emits a Hono app
  you `wrangler deploy` to Cloudflare, `vercel` to Vercel, or `docker
  build` for self-host; the same TS graph runs as notebook script,
  dev playground, and prod HTTP/SSE service.
- Built-in `Memory` (semantic + working + thread, pluggable across
  Postgres / Upstash / Pinecone / LibSQL) and `Eval` primitives mean
  the long-term-memory + observability story is in-house, not glued
  on.
