# assistant-ui

> Snapshot date: 2026-04. Upstream: <https://github.com/assistant-ui/assistant-ui>

"**Typescript/React Library for AI Chat.**" assistant-ui is a
React/TypeScript primitives library — Radix-/shadcn-style — for
building production-grade chat UIs in front of any agent backend.
Ships streaming, auto-scroll, attachments, markdown, code highlighting,
voice dictation, keyboard shortcuts, accessibility, and tool-call
rendering as composable pieces (`Thread`, `MessageList`, `Composer`,
`Toolbar`, `BranchPicker`, etc.) instead of one monolithic widget. The
CLI surface is `npx assistant-ui create` (scaffold) and
`npx assistant-ui init` (drop into an existing Next.js / Vite app).

## 1. Install footprint

- `npx assistant-ui create my-app` scaffolds a fresh Next.js project
  with `@assistant-ui/react`, a starter `Thread` component, the
  shadcn/ui theme, and a backend stub (AI SDK by default;
  flag-selectable LangGraph / Mastra / Assistant Cloud).
- `npx assistant-ui init` adds primitives to an existing repo —
  installs `@assistant-ui/react`, registers the shadcn theme, and
  drops generated components under `components/assistant-ui/`.
- `pnpm add @assistant-ui/react` for manual install. Backend bridges
  are separate packages: `@assistant-ui/react-ai-sdk`,
  `@assistant-ui/react-langgraph`, `@assistant-ui/react-mastra`,
  `@assistant-ui/react-edge`.
- React ≥ 18, TypeScript ≥ 5. Works under Next.js (App + Pages
  router), Remix, Vite, TanStack Start, Astro islands, plain CRA.
- Optional `@assistant-ui/react-markdown` (rehype/remark stack),
  `@assistant-ui/react-syntax-highlighter` (Shiki), and
  `@assistant-ui/react-data-stream` for the AI SDK v5 stream protocol.

## 2. Repo + version + license

- Repo: <https://github.com/assistant-ui/assistant-ui>
- Latest tag: **v0.1.4** (repo-level monorepo tag; per-package
  `@assistant-ui/react` publishes ≥10×/week to npm — pin individually)
- License: **MIT** —
  <https://github.com/assistant-ui/assistant-ui/blob/main/LICENSE>
- Default branch: `main`
- Language: TypeScript

## 3. Models supported

UI layer — model support is delegated to whichever backend bridge you
mount. With `@assistant-ui/react-ai-sdk` you inherit the full Vercel
AI SDK provider catalogue (OpenAI, Anthropic, Google Gemini, Mistral,
Perplexity, AWS Bedrock, Azure OpenAI, Cohere, Fireworks, Groq,
Together, Replicate, Hugging Face, Ollama, plus any
OpenAI-compatible endpoint). `@assistant-ui/react-langgraph` routes
through LangGraph's provider abstraction. `@assistant-ui/react-mastra`
inherits Mastra's provider list. The `Custom` runtime contract is a
single async generator returning text/tool-call deltas, so anything
that streams works.

## 4. MCP support

Indirect. assistant-ui itself does not speak MCP — it renders whatever
the backend streams. Pair with [`mcp-agent`](../mcp-agent/) /
[`mastra`](../mastra/) / a custom MCP-aware backend; tool calls and
elicitation prompts arrive as typed events that the library renders as
inline components via `assistant.tool()` registrations. The
`tool_calls` stream protocol matches the AI SDK shape exactly, so any
MCP-bridged AI SDK route renders for free.

## 5. Sub-agent model

Not a sub-agent framework. assistant-ui is the **view layer**. It
will faithfully render any multi-agent backend's stream — branching
threads (`BranchPicker`), human approval gates (`UIMessage.parts` with
`tool-input` / `tool-output` parts), and parallel agent timelines if
the backend emits them. Sub-agent orchestration belongs in the
backend ([`langgraph`](../langgraph/), [`mastra`](../mastra/),
[`crewai`](../crewai/), [`voltagent`](../voltagent/), etc.).

## 6. Telemetry stance

Off in OSS — the React library is client-side only and emits zero
network traffic on its own. The optional **Assistant Cloud** backend
(single env var `NEXT_PUBLIC_ASSISTANT_BASE_URL`) sends thread
persistence + analytics to assistant-ui's hosted service; opt-in only.
DeepWiki and Workweave badges in the upstream README are GitHub
analytics, not runtime telemetry.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **The shadcn/ui of AI chat.** Most React chat
libraries hand you a `<ChatBox />` and a theme prop; assistant-ui
hands you ~20 unstyled primitives (`Thread.Root`, `Thread.Viewport`,
`Composer.Input`, `Message.If user`, `BranchPicker.Previous`, etc.)
that compose into the ChatGPT UX in ~50 lines and into a
Perplexity / Claude / custom UX with the same primitives reordered.
Streaming, retries, attachments, voice dictation, keyboard shortcuts,
ARIA, RTL, and auto-scroll are handled at the primitive level so you
never re-implement them. Tool-call rendering is first-class:
`assistant.tool({ name, render })` registers a typed React component
that gets the live tool args + result. Backend-agnostic (AI SDK /
LangGraph / Mastra / custom) means the same UI survives a backend
swap.

**Weakness.** **It is a UI library, not a runtime.** Zero opinion on
agent orchestration, persistence, eval, or deployment — bring your
own backend or use Assistant Cloud (SaaS). Monorepo per-package
versioning with daily npm releases means dependency drift between
`@assistant-ui/react`, `@assistant-ui/react-ai-sdk`,
`@assistant-ui/react-markdown`, etc. is easy to hit; pin minors. The
public repo-level tags (v0.1.x) lag behind the actual npm package
versions (`@assistant-ui/react@0.x`) by a wide margin — use
`npm view @assistant-ui/react version` for the real number. Backed
by Y Combinator; the OSS / Cloud line is clearly drawn but you are
betting on a young company for the managed tier.

**When to choose.** You are **building** a TypeScript/React product
that needs a polished chat surface and you already have (or want to
build) the backend separately. Pair with
[`langgraph`](../langgraph/) / [`mastra`](../mastra/) /
[`voltagent`](../voltagent/) / [`pydantic-ai`](../pydantic-ai/) on the
agent side, or with [`mcp-agent`](../mcp-agent/) if you want
MCP-native orchestration. Skip if you want a terminal coding agent
(use [`opencode`](../opencode/) / [`claude-code`](../claude-code/) /
[`crush`](../crush/)) or a server-rendered chat (use
[`chainlit`](../chainlit/) / [`langflow`](../langflow/)). Skip if you
want a one-line `<Chat />` component — assistant-ui rewards
composition, not drop-in.
