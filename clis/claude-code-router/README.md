# claude-code-router

> Snapshot date: 2026-04. Upstream: <https://github.com/musistudio/claude-code-router>

"**Use Claude Code without an Anthropic account and route it to
another LLM provider.**" A local proxy + CLI (`ccr`) that intercepts
the API calls Anthropic's terminal coding agent makes and re-routes
them — model selection, request shape translation, per-tool model
override — to any OpenAI-compatible endpoint. The agent thinks it is
talking to Claude; in reality the planning step might be DeepSeek-V3,
the long-context step Gemini, the cheap edits a local Qwen-Coder via
Ollama. Includes a transformer registry so non-Claude providers see
the request shape they expect, and a small TUI / web UI for live
routing rules.

## 1. Install footprint

- `npm install -g @musistudio/claude-code-router` (or
  `pnpm add -g @musistudio/claude-code-router`).
- Node ≥ 18. Single npm package, three internal workspaces (`cli`,
  `server`, `ui`) wrapped together.
- After install: `ccr code` launches Anthropic's terminal coding
  agent through the local proxy; `ccr start` runs the proxy as a
  daemon; `ccr ui` opens the routing-rules dashboard at
  `http://127.0.0.1:3456`.
- Configuration lives in `~/.claude-code-router/config.json` —
  `Providers[]` declares endpoints (name + base URL + key + model
  list + transformer), `Router` declares the per-task default model
  (`default`, `background`, `think`, `longContext`, `webSearch`).

## 2. Repo + version + license

- Repo: <https://github.com/musistudio/claude-code-router>
- Latest tag: **v2.0.0**
- License: **MIT** —
  <https://github.com/musistudio/claude-code-router/blob/main/LICENSE>
- Default branch: `main`
- Language: TypeScript (Node)

## 3. Models supported

Any provider that speaks an OpenAI-compatible Chat Completions API:
OpenAI itself, Anthropic (passthrough), Google Gemini (AI Studio +
Vertex), DeepSeek (V3, R1, Coder), Qwen (DashScope + local), Moonshot
Kimi, Zhipu GLM, Groq, Mistral, Together, Fireworks, Cerebras,
SambaNova, OpenRouter (gateway to ~300 models), SiliconFlow, Volcengine
Ark, Bedrock-via-LiteLLM, Azure OpenAI, Ollama, vLLM, LM Studio,
llama.cpp, LiteLLM proxy. Per-route model override is the headline:
the long-context step, the planning step, the background-summary step,
and the web-search step can each pin a different provider + model in
`Router`.

## 4. MCP support

Inherited. The router does not implement MCP itself — it proxies
the upstream agent's traffic. Whatever MCP servers the wrapped agent
mounts continue to work transparently; the router only sits on the
LLM-call hop, not on the tool-call hop.

## 5. Sub-agent model

Inherited from the wrapped agent (Anthropic's terminal coding tool's
own sub-agent / `Task` mechanism). The router's contribution is
**routing per sub-agent role** — the planner can run on one model,
the implementer on another, the reviewer on a third, configured by
the agent's announced role / system prompt.

## 6. Telemetry stance

Off. No analytics in the OSS codebase. Egress is exactly the upstream
providers you declare in `config.json` plus the upstream agent's own
egress (e.g. its update channel). The proxy binds to `127.0.0.1` by
default; the optional UI is also localhost-only.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Per-task model routing for an agent harness
that hardcoded one provider.** Anthropic's terminal coding tool is a
deeply polished agent loop — sub-agents, hooks, skills, MCP — but
it natively only talks to Anthropic models. claude-code-router lifts
that constraint without touching the agent code: keep the UX, swap
the brain. The `Router` table lets you bind `default` → DeepSeek-V3
(cheap + fast for most edits), `think` → GPT-5 (hardest planning),
`longContext` → Gemini 2.5 Pro (1 M context for whole-repo reads),
`background` → local Qwen-Coder via Ollama (zero-cost summaries),
`webSearch` → Perplexity. The transformer registry rewrites request
shapes per provider (Gemini's `systemInstruction`, DeepSeek's
`reasoning_content`, OpenAI Responses API's `instructions`) so the
agent's standard tool-call protocol survives the swap. End result: the
best agent UX in the catalog, on whichever model is cheapest /
strongest / closest to your data, with a config-only switch.

**Weakness.** **You inherit two project's bugs and one project's
release cadence.** When the upstream agent ships a breaking change to
its prompt envelope or a new tool-call protocol, the router's
transformers need a matching update; until then, the new agent
version may misbehave on non-Anthropic providers. Quality will never
exceed the underlying model on the routed task — picking DeepSeek for
`think` on hard refactors is cheaper, *not* better. Some agent
features (Anthropic-specific extended-thinking budgets, prompt
caching headers, fine-grained vision IO) degrade or no-op against
providers that do not implement them. Operationally it is a long-
lived local proxy on `127.0.0.1` — one more daemon to babysit.

**When to choose.** You already use Anthropic's terminal coding agent
and want to **mix providers per task** without leaving its UX —
either to cut spend (cheap model for background work), to reach
context lengths or capabilities Claude does not have (Gemini 2.5 Pro's
1 M context, DeepSeek R1's reasoning trace), to comply with data-
residency rules (route certain repos through a self-hosted Qwen on
Ollama), or simply to evaluate other models inside the same agent
harness. Pair with [`kit`](../kit/) to ship the routed local model as
a versioned ModelKit. Skip if you do not use the upstream agent (the
router has no value standalone), if you need the *exact* Anthropic-
specific features (use the agent directly), or if you want an agent
that natively multi-provides (use [`opencode`](../opencode/),
[`forge`](../forge/), [`crush`](../crush/), or [`gemini-cli`](../gemini-cli/)).
