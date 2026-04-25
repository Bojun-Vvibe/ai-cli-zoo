# rivet

> Snapshot date: 2026-04. Upstream: <https://github.com/Ironclad/rivet>
> License file: <https://github.com/Ironclad/rivet/blob/main/LICENSE>
> Pinned: `app-v1.11.3` (2025-08-08, desktop app). Library packages
> (`@ironclad/rivet-core`, `@ironclad/rivet-node`) ship from the
> same monorepo on a parallel cadence. Default branch is `main`.

An **open-source visual editor for LLM graphs** — a desktop IDE
(Tauri / Electron-style) where prompt chains, tool calls, branching,
sub-graphs, and embedded code nodes are wired together as a
DAG, then exported as a JSON graph that an embedded runtime
(`@ironclad/rivet-node`) executes inside a Node service. The
contrast point against [`langflow`](../langflow/) and `flowise` is
"author in the desktop IDE, ship the graph as a versioned artifact
your backend embeds" rather than "author and host inside a hosted
web canvas". The contrast against code-first frameworks like
[`langgraph`](../langgraph/) or [`mastra`](../mastra/) is the
visual canvas itself: non-engineers (PMs, prompt engineers, ops)
can edit a Rivet graph without touching TypeScript.

## 1. Install footprint

- **Desktop IDE**: download the platform installer from
  <https://rivet.ironcladapp.com/> or the GitHub Releases page —
  signed `.dmg` (macOS), `.msi` (Windows), `.AppImage` (Linux),
  ~150 MB.
- **Node embedding**: `npm install @ironclad/rivet-node` (~5 MB);
  load a `.rivet-project` file with `loadProjectFromFile(...)` and
  call `runGraph(processor, projectFile, graphId, inputs)`.
- **Browser embedding**: `@ironclad/rivet-core` is the runtime
  without Node-only nodes (no filesystem / shell), suitable for
  `@ironclad/rivet-web` in the browser.
- **CLI / headless run**: no official CLI — the standard pattern is
  `node run-graph.ts` against the embedded runtime; community
  wrappers exist (`rivet-cli`) but are not in the upstream repo.

## 2. License

MIT (file is `LICENSE` at the repo root,
<https://github.com/Ironclad/rivet/blob/main/LICENSE>). All packages
in the monorepo (`core`, `node`, `app`, plugin SDK) ship MIT. There
is no commercial / managed Rivet — Ironclad uses it internally and
publishes it as OSS; the project has no upsell tier.

## 3. Models supported

Built-in Chat nodes target OpenAI, Anthropic, Google (Gemini),
Mistral, Cohere, Groq, Ollama, and any OpenAI-compatible base URL
(so [`vllm`](../vllm/), [`localai`](../localai/),
[`llamafile`](../llamafile/), and [`litellm`](../litellm/) proxies
all work). The plugin system (`createPlugin`) lets you register
custom node types — most teams add an org-internal "Call our
inference proxy" node so credentials never live in the graph file.
Embedding nodes target OpenAI, Cohere, and any HF inference
endpoint.

## 4. MCP support

No first-party MCP server or client. The plugin SDK is the
extension surface — a community MCP-tool node would be a
straightforward implementation (call `tools/list` once at graph
load, surface each tool as an MCP-Call node), but is not in the
upstream tree as of this snapshot. The inverse direction —
exposing a Rivet graph as an MCP tool — is one `@ironclad/rivet-
node` wrapper around the reference MCP server SDK.

## 5. Sub-agent model

**Sub-graphs are the sub-agent unit**. A Subgraph node references
another graph in the same project by ID, passes typed inputs in,
and reads typed outputs back; recursion is allowed (with depth
limit) so a "router → specialist sub-graphs" pattern is one wire.
Loop nodes iterate sub-graphs over arrays. The Code node embeds
JS / TS for arbitrary glue logic between LLM calls. Branching is
explicit (`If` / `Match` nodes) — no implicit control flow, which
is the thing that makes Rivet graphs auditable.

## 6. Telemetry stance

The desktop app **does not phone home** by default — no built-in
analytics in the OSS build. API keys for providers live in the
local app's secrets store (OS keychain) and are never serialised
into `.rivet-project` files (which are shareable JSON). When
embedded with `@ironclad/rivet-node`, the runtime makes no network
calls beyond the LLM provider URLs your graph nodes specify.
Tracing is opt-in via the `onProcessEvent` hook on `runGraph` —
common pattern is to bridge those events into your existing
observability stack ([`langfuse`](../langfuse/),
[`arize-phoenix`](../arize-phoenix/), or OpenTelemetry).

## 7. Prompt-cache strategy

None at the runtime layer. Caching is provider-side or via a
proxy node — drop a [`litellm`](../litellm/) proxy in front and
let it cache, or add a `Cache Lookup` Code node that hashes the
prompt + reads/writes Redis before the Chat node. Anthropic
prompt-cache markers can be passed through via the Chat node's
`extraBody` field.

## 8. Hot keybinds

The desktop IDE is the everyday surface. Shortcuts are macOS-style
(Cmd-* on macOS, Ctrl-* elsewhere):

| Action                  | Keybind     |
| ----------------------- | ----------- |
| New node (palette)      | `Space`     |
| Run current graph       | `Cmd-Enter` |
| Save project            | `Cmd-S`     |
| Toggle data inspector   | `Cmd-I`     |
| Frame selection         | `F`         |
| Duplicate node          | `Cmd-D`     |
| Disable/enable node     | `Cmd-/`     |

Embedded run shape:

```ts
import { loadProjectFromFile, runGraph, NodeDatasetProvider } from "@ironclad/rivet-node";

const project = await loadProjectFromFile("./pipelines/triage.rivet-project");
const result = await runGraph(project, {
  graph: "main",
  inputs: {
    ticket: { type: "string", value: "Customer can't reset password..." },
  },
  context: {
    settings: { openAiKey: process.env.OPENAI_API_KEY },
  },
  onProcessEvent: {
    nodeFinish: (e) => log.info({ node: e.node.title, ms: e.duration }),
  },
});
console.log(result.classification.value, result.suggested_reply.value);
```

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A desktop visual editor that produces a
versioned, embeddable graph artifact** — non-engineers can edit
prompt chains in the IDE, the resulting `.rivet-project` lives in
git as plain JSON, and the Node service ships
`@ironclad/rivet-node` to execute it. No hosted runtime
dependency, no separate state store, no "the graph lives in a
SaaS account" lock-in.

**Weakness.** The desktop-first authoring model means **you cannot
edit a graph from a browser without standing up the experimental
`@ironclad/rivet-web` host**, and the desktop app release cadence
(roughly quarterly) is slower than the SDK. The plugin ecosystem
is small compared to LangChain/LangGraph — most integrations you
need to write yourself. Headless CI execution works but lacks an
official CLI binary; teams script around `tsx run.ts`.

**When to choose.**

- You want **non-engineers to author and review prompt graphs**
  with diffs you can read in a PR (the JSON is structured but
  reviewable; the IDE renders it visually).
- You want a **graph artifact that ships inside your existing Node
  service**, not a hosted execution environment.
- You want **auditable explicit control flow** (If / Match /
  Subgraph / Loop) instead of implicit agent-decides-next-step
  semantics.

**When not to choose.** You want a **code-first TypeScript agent
framework** with type-checked nodes and IDE refactors —
[`mastra`](../mastra/) or [`langgraph`](../langgraph/) are closer.
You want a **hosted no-code canvas** with auth, sharing, and
deploy in one product — [`langflow`](../langflow/) (Python) or
Flowise (Node) fit that shape, at the cost of running their
server. You're a **Python shop** — Rivet is TS/Node-only and
calling it from Python means an HTTP boundary.
