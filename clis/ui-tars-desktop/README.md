# ui-tars-desktop

> Snapshot date: 2026-04-26. Upstream: <https://github.com/bytedance/UI-TARS-desktop>

"**The open-source multimodal AI agent stack: connecting
cutting-edge AI models and agent infra.**" UI-TARS-desktop is the
ByteDance / Seed-TARS umbrella repo that ships two related
agents: **Agent TARS CLI** — a terminal-first multimodal agent
that wires LLMs + GUI Agent + Vision into shell + browser +
MCP-tool execution — and **UI-TARS Desktop** — an Electron
application that turns the dedicated open-weight UI-TARS-1.5
vision-grounding model into a click-and-type operator for the
local computer or a remote VM / browser.

## 1. Install footprint

- Agent TARS CLI: `npm install -g @agent-tars/cli` (or
  `pnpm add -g @agent-tars/cli`); launch with
  `agent-tars --provider anthropic --model claude-sonnet-4` or
  any OpenAI-compatible endpoint via `--base-url`. Boots a
  terminal session plus an optional Web UI at
  `http://localhost:8888` for stream / event inspection.
- UI-TARS Desktop app: download the prebuilt `.dmg` / `.exe` /
  `.AppImage` from the GitHub Releases page, or
  `pnpm install && pnpm run dev:ui-tars` from a clone for the
  developer build.
- AIO Sandbox (recommended for Agent TARS v0.3+): one Docker
  image that bundles shell + browser + file tools as an isolated
  execution environment the agent talks to over MCP, so tool
  calls never touch the host.
- SDK: `@ui-tars/sdk` (npm) is a cross-platform toolkit for
  building your own GUI-automation agent on top of the
  UI-TARS-1.5 grounding model + a planner of your choice.

## 2. Repo + version + license

- Repo: <https://github.com/bytedance/UI-TARS-desktop>
- Latest release: **v0.3.0** (Agent TARS CLI, published
  2025-11-04); UI-TARS Desktop application currently at v0.2.x
  (Remote Computer / Browser Operator).
- License: **Apache-2.0** —
  <https://github.com/bytedance/UI-TARS-desktop/blob/main/LICENSE>
- Default branch: `main`
- Language: TypeScript (Electron + Node CLI), with Python helper
  scripts for model deployment.
- Stars: ~29.5k

## 3. Models supported

- Agent TARS planner: any chat model reachable via the OpenAI
  Chat Completions / Anthropic Messages / Gemini API surface —
  GPT-4o / 4.1 / o-series, Claude 3.5 / 3.7 / 4 Sonnet, Gemini
  2.x, plus any OpenAI-compatible endpoint (vLLM, SGLang,
  Together, Groq, OpenRouter, Ollama, LiteLLM).
- Visual grounding: **UI-TARS-1.5-7B** (open weights, served
  via vLLM / SGLang / Hugging Face Inference / ModelScope), or
  any vision-capable chat model fed the screenshot directly
  (Claude Sonnet, GPT-4o) when no dedicated grounder is
  available.
- Tool surface: shell, file I/O, browser (CDP), MCP client for
  any third-party MCP server, plus the AIO Sandbox bundle as a
  single MCP endpoint that fronts shell + browser + filesystem
  for safe in-container execution.

## 4. Notable angle

**Open-weight visual-grounding model + a CLI-first agent loop in
the same repo.** The catalog already has DOM-based browser agents
([`browser-use`](../browser-use/), [`stagehand`](../stagehand/),
[`skyvern`](../skyvern/)) and a desktop-pixel agent
([`agent-s`](../agent-s/)), but UI-TARS-desktop is the only entry
where ByteDance ships the **grounding model itself** as part of
the stack: UI-TARS-1.5-7B is the open-weight vision-language model
trained specifically to translate "click the blue Submit button
near the bottom" into pixel coordinates, and the desktop app +
Agent TARS CLI are reference clients that drive it. That makes it
the cheapest way in the catalog to stand up a fully-open
"computer-using agent" — planner of your choice, grounder served
by your own vLLM / SGLang on a single 24 GB GPU, no closed
frontier API on the grounding hop. The Agent TARS CLI half is the
terminal-native counterpart to OpenAI's Operator and Anthropic's
Computer Use: a multimodal planner + MCP tool host + Web UI
event-stream viewer that runs the same loop on your laptop, with
the AIO Sandbox containing tool execution. Trade-offs: pixel-
grounded agents are still slower per step than DOM-grounded ones
(Agent TARS shines for native apps + Canvas-rendered UIs where
[`browser-use`](../browser-use/) can't read the DOM); release
cadence is fast and breaking; Web UI is optional but expected.
Use it when you want an **open-weight** computer-using agent end-
to-end; pick [`browser-use`](../browser-use/) when DOM scraping
will do; pick [`agent-s`](../agent-s/) when you want the academic
GUI-agent baseline with OSWorld evaluation harness.

## 5. Last verified

2026-04-26 via `gh api repos/bytedance/UI-TARS-desktop` and
`gh api repos/bytedance/UI-TARS-desktop/releases/latest` →
Agent TARS CLI v0.3.0 (2025-11-04), license Apache-2.0 at
<https://github.com/bytedance/UI-TARS-desktop/blob/main/LICENSE>.
