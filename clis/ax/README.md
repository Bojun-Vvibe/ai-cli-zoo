# ax

> Snapshot date: 2026-04. Upstream: <https://github.com/ax-llm/ax>

"**The pretty much 'official' DSPy framework for TypeScript.**" Ax is
a **TypeScript port of the DSPi signature / module / optimizer
pattern** from Stanford's [`dspy`](../dspy/) project: instead of
hand-tuning prompts, you declare a typed *signature*
(`question:string -> answer:string, confidence:number`), let the
framework synthesize the prompt, run it, and optionally let an
optimizer (MIPROv2, BootstrapFewShot) compile the program against a
metric on a labeled dataset. The result is a reproducible,
type-checked LLM pipeline that you can `git diff` like normal code,
plus a CLI for running and evaluating Ax programs without writing a
host process.

## 1. Install footprint

- Library: `npm install @ax-llm/ax` (or `pnpm add`, `yarn add`).
- CLI: `npm install -g @ax-llm/ax-cli` (alternatively
  `npx @ax-llm/ax-cli`).
- Node ≥ 20. ESM + CJS dual build.
- Library entry points: `ai({ name: "openai", apiKey, config })`,
  `ax\`question:string -> answer:string\`` (signature literal),
  `AxGen`, `AxAgent`, `AxFlow`, `AxMCPClient`, `AxOptimizer`.
- CLI surface (`ax …`): `ax run <signature.ts>`, `ax eval`,
  `ax optimize`, `ax mcp serve`, `ax mcp call`.
- Configure via env: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  `GOOGLE_API_KEY`, `GROQ_API_KEY`, etc., or per-call config
  passed to `ai()`.

## 2. Repo + version + license

- Repo: <https://github.com/ax-llm/ax>
- Latest release: **v20.0.0** (2026-04-25)
- License: **Apache-2.0** —
  <https://github.com/ax-llm/ax/blob/main/LICENSE>
- Default branch: `main`
- Language: TypeScript

## 3. Models supported

OpenAI (Chat + Responses), Anthropic, Google Gemini (AI Studio +
Vertex), Cohere, Mistral, Groq, Together, Ollama, AWS Bedrock,
Hugging Face, DeepSeek, xAI Grok, OpenRouter, any OpenAI-compatible
endpoint via the OpenAI provider with `apiURL` override. Multi-model
routing in one program is first-class — assign different models to
different `AxGen` modules in the same flow.

## 4. MCP support

Yes — both client and server. `AxMCPClient` mounts an MCP server's
tools as Ax tools usable inside any signature; `AxMCPServer` /
`ax mcp serve` exposes an Ax program as an MCP server consumable
from `claude-code` / `opencode` / `goose` / Cursor. One of the few
TS-native frameworks in the catalog with bidirectional MCP.

## 5. Sub-agent model

`AxAgent` wraps a signature plus tools as a self-contained agent;
`AxFlow` composes agents and `AxGen` modules into a typed pipeline
(branches, loops, parallel, conditional). Sub-agents are explicit:
one agent's output type must match the next stage's input type, the
TypeScript compiler enforces this at build time. No auto-spawn.

## 6. Telemetry stance

Off by default. OpenTelemetry-native: every `AxGen` call emits spans
with model name, token counts, latency, and tool-call traces; you
attach an exporter (Phoenix / Langfuse / Logfire / OTel collector)
if you want them shipped, otherwise they go nowhere. No first-party
analytics dashboard. Egress is exactly the LLM provider you
configure plus any tools your program calls.

## 7. Killer commands / API

```ts
import { ai, ax } from "@ax-llm/ax";

const llm = ai({ name: "openai", apiKey: process.env.OPENAI_API_KEY! });

// signature-as-program: declared shape, synthesized prompt
const summarize = ax`document:string -> summary:string, key_points:string[]`;

const { summary, key_points } = await summarize.forward(llm, {
  document: longText,
});
```

```sh
ax run ./programs/summarize.ts --input doc.txt   # one-shot exec
ax eval ./programs/summarize.ts --dataset eval.jsonl --metric f1
ax optimize ./programs/summarize.ts --optimizer mipro_v2 --trials 50
ax mcp serve ./programs/summarize.ts --port 8765
```

## 8. Killer feature, weakness, when to choose

**Killer feature.** **Typed DSPy-style signatures + a real optimizer
in TypeScript, with bidirectional MCP.** The signature literal
(`` ax`question:string -> answer:string, confidence:number` ``) is
both a runtime program and a compile-time type — your IDE knows the
output shape, the framework synthesizes the prompt, and an optimizer
can compile the program against a metric on a labeled dataset
(MIPROv2, BootstrapFewShot, custom). Pair that with first-class
MCP-server output (`ax mcp serve`) and an Ax program becomes a
versionable, optimizable, type-checked tool that any MCP-aware
coding agent can consume — a shape no other TS framework in the
catalog combines.

**Weakness.** Fast-moving (`v20.0.0` shipped 2026-04-25; major
versions break the API every few months) — pin exact versions in
`package.json` and budget for migration when you upgrade. The
optimizer story works best with structured tasks (classification,
extraction, multi-step QA) and a real labeled dataset; for
free-form generation tasks the optimizer's signal is weak. CLI
surface is genuinely a CLI, not a chat REPL — no `aichat`-style
interactive mode. Smaller community than [`dspy`](../dspy/) Python,
so framework-specific Stack Overflow answers are sparse.

**When to choose.** You are writing a TypeScript service or Edge
function that calls LLMs and you want the call to look like a typed
function with a synthesized prompt, not a templated string — and you
want the option to *optimize* the prompt against a metric instead of
hand-tuning it forever. Pair with [`pydantic-ai`](../pydantic-ai/)
on the Python side if you operate a polyglot team that wants the same
typed-LLM pattern in both languages, or with
[`promptfoo`](../promptfoo/) for evaluation harness. Skip if you are
in Python (use [`dspy`](../dspy/) directly), if you want a coding
agent that edits files ([`opencode`](../opencode/),
[`claude-code`](../claude-code/)), or if your team is allergic to
ESM-only TypeScript libraries.
