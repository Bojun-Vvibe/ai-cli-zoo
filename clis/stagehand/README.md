# stagehand

> Snapshot date: 2026-04. Upstream: <https://github.com/browserbase/stagehand>
> License file: <https://github.com/browserbase/stagehand/blob/main/LICENSE>
> Pinned: `stagehand-server-v3/v3.6.3` (2026-03-31, monorepo tag).
> Default branch is `main`. HEAD verified at
> `6b9b46d81e32ed475772bd6c2299782985eef65a`. Install as
> `npm install @browserbasehq/stagehand` (TypeScript) or
> `pip install stagehand-py` (Python).

An **AI browser-automation SDK** that wraps Playwright with three
LLM-driven primitives — `act("click the cart icon")`,
`extract({schema, instruction})`, `observe("what can I click on
this page?")` — plus a thin atomic layer (`page.goto`, `page.locator`,
…). The contrast against [`browser-use`](../browser-use/) is "you
own the script and reach for the LLM only when a step is fragile",
not "the LLM drives the whole loop", which makes Stagehand a better
fit for **deterministic, version-controlled browser flows that need
a sprinkle of intelligence** (e.g. a checkout regression suite that
self-heals when the cart button moves, but whose order of operations
is hand-pinned).

## 1. Install footprint

- **TypeScript**: `npm install @browserbasehq/stagehand` (~12 MB
  with peer deps — `playwright`, `zod`, `openai`, `@anthropic-ai/sdk`).
  Drop-in replacement for `chromium.launch()` via
  `new Stagehand({ env: "LOCAL" | "BROWSERBASE", modelName, ... })`.
- **Python**: `pip install stagehand-py` (~25 MB with deps —
  `playwright`, `pydantic`, `httpx`, `openai`, `anthropic`).
- **Browser**: `npx playwright install chromium` for local mode;
  Browserbase managed cloud for remote (`env: "BROWSERBASE"`,
  `BROWSERBASE_API_KEY` + `BROWSERBASE_PROJECT_ID`).
- **CLI**: `npx stagehand init` scaffolds a TypeScript starter
  (`stagehand.config.ts`, sample script, `package.json` with
  `tsx` runner); `npx create-browser-app` is the higher-level
  scaffolder that picks template + provider + env mode interactively.
- **MCP server**: `npx @browserbasehq/mcp-stagehand` — published
  alongside the SDK, exposes `act` / `extract` / `observe` /
  `navigate` / `screenshot` as MCP tools so any MCP-aware host
  ([`opencode`](../opencode/), [`claude-code`](../claude-code/),
  [`continue`](../continue/), [`cline`](../cline/)) can drive a
  browser without bundling Playwright.

## 2. License

MIT (file is `LICENSE` at the repo root,
<https://github.com/browserbase/stagehand/blob/main/LICENSE>). The
SDK runs entirely against your own LLM API keys + a local Chromium
in `env: "LOCAL"`, so the OSS path needs no Browserbase account. The
managed cloud (Browserbase) is the commercial offering — it adds
session recording, residential proxies, captcha solving, and a
remote-debug surface, but is optional.

## 3. Models supported

Anthropic (Claude 3.5 / 3.7 / 4 Sonnet recommended for `act` /
`observe`), OpenAI (gpt-4o / gpt-4o-mini / o-series), Google Gemini
(2.0 Flash / 2.5 Pro), Groq, Cerebras, Together, any LiteLLM-routed
provider via `modelClientOptions: { baseURL, apiKey }`. The DOM-
extraction prompt is tuned for instruction-following models with
JSON-mode or tool-calling — small open models that lack
function-calling will degrade `extract` quality. Model selection is
per-method (`page.act("...", { modelName: "claude-3-7-sonnet-20250219"
})`) so cheap models can drive `observe` while a stronger model
runs `extract` against the same page.

## 4. MCP support

**First-class** — the project ships `@browserbasehq/mcp-stagehand`,
a reference MCP server that exposes the four primitives + `goto` +
`screenshot` + `wait` as MCP tools. The intended pattern is "any
MCP-aware coding agent gets browser-automation as a tool by adding
one stdio MCP server entry to its config". The inverse direction —
calling MCP tools *from inside* a Stagehand script — is the host's
responsibility (Stagehand is a Playwright wrapper, not an MCP
client), but the typical wiring is "agent → MCP → Stagehand →
Chromium" with the agent retaining orchestration.

## 5. Sub-agent model

Three composable primitives, no scheduler:

- `page.act(instruction)` — LLM picks a DOM action (click, type,
  scroll, select) given the natural-language step; the SDK previews
  the planned action via `observe()` first to keep cost bounded
  and guard against ambiguity.
- `page.extract({ schema: z.object({...}), instruction })` — LLM
  returns Zod / Pydantic-shaped data extracted from rendered DOM +
  page-text; structured-output mode under the hood, so the result
  is type-safe.
- `page.observe(instruction)` — returns a list of candidate
  actionable elements + their selectors + suggested next steps,
  used either as a standalone "what's on this page?" query or as
  a planning step before `act`.

Multi-agent / parallel-page browsing is achieved by spawning N
`Stagehand` instances; there's no built-in supervisor. Pair with
[`langgraph`](../langgraph/) or a plain async `Promise.all` for
fan-out crawls.

## 6. Telemetry stance

OSS SDK does not phone home in `env: "LOCAL"` mode beyond the
configured LLM provider URL. `env: "BROWSERBASE"` routes all browser
sessions through `api.browserbase.com` (commercial managed
infrastructure) — opt-in by setting `BROWSERBASE_API_KEY`. The SDK
emits structured logs via `logger` — pipe to
[`langfuse`](../langfuse/) / [`opik`](../opik/) /
[`arize-phoenix`](../arize-phoenix/) by attaching an OTel
exporter; no built-in tracing UI.

## 7. Prompt-cache strategy

`act` and `observe` reuse a cached DOM-summary representation across
calls within the same `page` lifetime — the SDK builds a compressed
accessibility-tree-flavoured representation once per navigation and
re-uses it for subsequent prompts on the same URL until the URL or
DOM mutation count exceeds a threshold. Anthropic prompt caching is
enabled by default for the system prompt + DOM context when using
Claude models. There's no on-disk cache — every fresh `Stagehand`
re-summarises from scratch.

## 8. Hot keybinds

No TUI. The surface is a Playwright-shaped script:

```ts
import { Stagehand } from "@browserbasehq/stagehand";
import { z } from "zod";

const stagehand = new Stagehand({
  env: "LOCAL",
  modelName: "claude-3-7-sonnet-20250219",
  modelClientOptions: { apiKey: process.env.ANTHROPIC_API_KEY },
});
await stagehand.init();

const page = stagehand.page;
await page.goto("https://news.ycombinator.com");

const top = await page.extract({
  instruction: "the top 3 story titles, points, and submitter",
  schema: z.object({
    stories: z.array(z.object({
      title: z.string(),
      points: z.number(),
      submitter: z.string(),
    })),
  }),
});

await page.act("click the comments link of the top story");
const comments = await page.extract({
  instruction: "the first 5 top-level comments",
  schema: z.object({
    comments: z.array(z.object({ author: z.string(), body: z.string() })),
  }),
});

await stagehand.close();
console.log({ top, comments });
```

`npx tsx script.ts` runs it.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Three small, composable LLM primitives layered
over plain Playwright** — your script reads like ordinary browser
automation (`page.goto`, `page.locator`) until you hit a step that
should self-heal across UI redesigns, and then `page.act("click the
checkout button")` does the LLM-driven thing. The `observe → act`
preview pattern keeps token cost bounded and gives the script a
deterministic dry-run mode for CI. Combined with the published MCP
server, any MCP-aware coding agent gains a production-grade browser
tool with one config line.

**Weakness.** The structured-output paths (`extract`,
`observe`) lean on JSON-mode / tool-calling, so small open models
that don't function-call degrade noticeably; the recommended
matrix is Claude / GPT-4o / Gemini 2.x. `act` failures on
heavily-iframe / Shadow-DOM apps still need manual fallbacks
(`page.locator(...)`) — the LLM doesn't magically resolve
cross-frame selectors. The Browserbase managed product is the
revenue surface, so the docs lean toward `env: "BROWSERBASE"`;
local mode is fully supported but you'll skim past Browserbase
sections.

**When to choose.**

- You have **Playwright-shaped browser flows** that break on UI
  redesigns and need **per-step LLM self-healing** without rewriting
  the whole loop as agent-driven.
- You want to **expose a browser to an MCP-aware coding agent** —
  the published MCP server is the shortest path.
- You need **typed extraction** from rendered web pages with Zod /
  Pydantic schemas, not "scrape HTML and post-parse".

**When not to choose.** You want **the LLM to plan and drive the
entire browser session end-to-end** — [`browser-use`](../browser-use/)
is the agent-first choice. You only need DOM scraping (no JS, no
auth) — `httpx` + `selectolax` / `lxml` is faster and cheaper. You
want a **headless web research agent with citations** —
[`gpt-researcher`](../gpt-researcher/) / [`khoj`](../khoj/) layer
research / retrieval on top of browsing. You want a **fully
managed cloud browser** without writing scripts — Browserbase's
managed product (or commercial peers) is the SaaS path; Stagehand
is the SDK that powers it.
