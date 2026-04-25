# browser-use

> Snapshot date: 2026-04. Upstream: <https://github.com/browser-use/browser-use>
> License file: <https://github.com/browser-use/browser-use/blob/main/LICENSE>
> Pinned: **0.12.6** (2026-04-02, PyPI: `browser-use`).

`browser-use` is a Python library + CLI that gives an LLM agent direct
control of a real Chromium browser via Playwright, with a structured
DOM-extraction layer that turns each page into a numbered list of
clickable / typeable / scrollable elements the model can address by
index. The agent loop is "screenshot + numbered DOM tree + URL +
last-action result → next action(s) → execute → observe", and the
action vocabulary is deliberately small: `click`, `type`, `scroll`,
`go_to_url`, `extract`, `done`. Distinguishing trait against earlier
LLM-browser projects (Selenium-AI, AutoGPT browser plugins): the
DOM-to-prompt encoding is engineered for *small* model context
windows — only interactive elements get indexed, hidden / decorative
nodes are filtered out, and the screenshot is annotated with the same
indices so vision and DOM agree on what "element 23" means.

## 1. Install footprint

- `pip install browser-use` (Python ≥ 3.11). Pulls Playwright,
  Pydantic 2, LangChain providers (OpenAI / Anthropic / Gemini /
  AWS / Azure / DeepSeek / Groq / Ollama), `posthog`, `mem0ai`,
  `lmnr` (Laminar tracing). First install ≈ 200 MB; you also need
  `playwright install chromium` (≈ 150 MB more).
- CLI entry point: `browser-use` (and alias `browseruse`) — Typer
  app at `browser_use.skill_cli.main:main`. Subcommands include
  `browser-use run "<task>"` for one-shot tasks and
  `browser-use chat` for an interactive REPL with a persistent
  Chromium session.
- Also ships a Python SDK (`from browser_use import Agent`) which
  is how most users consume it; the CLI is the quick-try surface.
- Hosted control plane at `cloud.browser-use.com` is opt-in (set
  `BROWSER_USE_API_KEY`) — useful for running headed browsers on
  someone else's infra; the OSS CLI runs fully local.

## 2. Repo + version + license

- Repo: <https://github.com/browser-use/browser-use>
- Latest release: **0.12.6** (2026-04-02)
- License: **MIT** —
  <https://github.com/browser-use/browser-use/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (+ TypeScript bits for the DOM-extraction script
  injected into the page)

## 3. Models supported

LLM-agnostic via LangChain's chat-model adapters: OpenAI (GPT-4.1 /
GPT-4o / o-series, Responses API), Anthropic (Claude Sonnet / Opus /
Haiku, with computer-use tool variant), Google Gemini (AI Studio +
Vertex), AWS Bedrock, Azure OpenAI, DeepSeek (V3, R1), Groq (Llama,
Qwen, Kimi-K2), Ollama (local). The agent is multi-modal — it
benefits substantially from vision (the annotated screenshot is the
disambiguator when the DOM tree is ambiguous), so vision-capable
models (GPT-4.1, Claude Sonnet, Gemini 2.x) are the default
recommendation; pure-text models work but degrade on JS-heavy SPAs.

## 4. MCP support

**Yes — both client and server.** Server: `browser-use --mcp` exposes
the agent's browser primitives (`navigate`, `click`, `type`,
`extract_text`, `screenshot`, `scroll`) as MCP tools, so any MCP-aware
host (Claude Desktop, [opencode](../opencode/),
[goose](../goose/), [crush](../crush/)) can hand a browser to its own
LLM without bringing in `browser-use`'s agent loop. Client: the
`browser-use` agent itself can mount external MCP servers as
additional tools (filesystem, shell, search) so a single task can
mix browsing and local actions.

## 5. Sub-agent model

**Yes — first-class via the `Controller` + `Agent` split.** A parent
agent can spawn child agents that each get their own browser context
(separate cookies, separate tab, separate viewport) by calling
`Controller.run_subtask(...)`; results are returned as structured
output to the parent. The catalog use case is "open 30 product pages
in parallel, extract price + stock, return a table" — fan-out is at
the browser-context level, so child agents don't fight for the same
DOM. No persistent state between runs (no memory daemon); the
optional `mem0ai` integration adds a vector-store memory layer if
you want one.

## 6. Telemetry stance

**On by default, opt-out.** PostHog (`posthog>=6.7.0`) ships anonymous
usage events (CLI invocations, model name, success / failure, no
prompt content); Laminar (`lmnr>=0.6.31`) is wired for opt-in trace
export to Laminar Cloud. Disable with `ANONYMIZED_TELEMETRY=false`
in the environment, or build a slim install without the `posthog`
dep. The browser itself is the network surface that matters — every
page the agent visits sees its IP and User-Agent; route through a
proxy (`browser-use --proxy http://...`) if that's a concern. Cloud
sync to `cloud.browser-use.com` requires explicit `BROWSER_USE_API_KEY`.

## 7. Prompt-cache strategy

Anthropic ephemeral cache wired on the system prompt + the static
parts of the DOM-extraction prompt (action schema, output format) so
the per-step cost on a 30-step browsing trajectory is dominated by
the new DOM tree + screenshot, not the boilerplate. OpenAI and
Bedrock rely on automatic provider-side prefix cache. The screenshot
is *not* cached (different per step by design); the DOM tree changes
between steps but its leading frame (URL bar, top nav) often doesn't,
which the provider-side prefix cache picks up automatically.

## 8. Hot keybinds (CLI / chat REPL)

| Key / command | Action |
|---------------|--------|
| `Enter` | Send turn / start task |
| `Ctrl+C` | Cancel current step (browser stays open) |
| `Ctrl+D` | Exit (closes browser) |
| `/screenshot` | Print current page screenshot to terminal (iTerm2 / Kitty inline) |
| `/dom` | Print numbered DOM tree of the current page |
| `/back` | Browser back |
| `/url <url>` | Navigate the current tab |
| `/save <name>` | Save the current trajectory as a replayable JSON |

For the one-shot `browser-use run "<task>"` form, useful flags:

- `--headless` — run without a visible window (default is headed so
  you can watch)
- `--max-steps 30` — cap the agent loop
- `--llm gpt-4.1` / `--llm claude-sonnet-4.5` — pick the model
- `--save-recording out.gif` — record a screen capture of the run

## 9. Killer feature, weakness, when to choose

- **Killer:** **Annotated DOM + annotated screenshot share an index
  space, so vision and structure agree.** Earlier LLM-browser tools
  forced you to pick one (pure-DOM AutoGPT plugins drowned in
  `<div>` noise; pure-vision SeeAct couldn't reliably click hidden
  buttons). `browser-use` numbers every interactive element in *both*
  the DOM string fed to the model *and* the screenshot overlay, so
  the model can resolve "element 23 is the blue 'Buy now' button"
  cross-modally. The result is the highest WebArena / Mind2Web
  benchmark numbers of any open-source browser agent as of writing,
  and — more practically — it actually completes "log in to my
  Costco account and reorder the toilet paper" without manual
  babysitting.
- **Weakness:** **Real browsers are slow and expensive.** Each step
  is a Playwright round-trip + a vision-model call; a 20-step task
  on Claude Sonnet costs ~$0.30 and takes 60–90 seconds. Parallel
  fan-out via sub-agents helps but multiplies cost. The library is
  also tightly coupled to LangChain's chat-model abstraction (which
  brings its own heavy dep tree and occasional API drift) and to
  Chromium specifically (Firefox / WebKit work but the DOM-extraction
  script is tuned for Chromium quirks). Captcha / WAF defences treat
  it like any other automation — Cloudflare Turnstile blocks it
  reliably, so heavily-protected sites need a residential-proxy
  layer the project does not provide.
- **Choose it when:** the task lives behind a UI with no usable API
  — internal admin consoles, legacy SaaS dashboards, vendor portals,
  account-management flows. Pair with [`crewai`](../crewai/) or
  [`pydantic-ai`](../pydantic-ai/) when the browsing step is one
  tool inside a larger agent, and with [`e2b`](../e2b/) if you need
  to run the browser in a sandboxed cloud VM rather than on the
  agent host. Skip if the same data is reachable via REST or
  scraping — Playwright + a parser is an order of magnitude cheaper
  per page than the LLM loop. The catalog's clearest answer to
  "give my agent hands on the web" without committing to a vendor's
  hosted browser-as-a-service.
