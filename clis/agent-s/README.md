# agent-s

> Snapshot date: 2026-04-26. Upstream: <https://github.com/simular-ai/Agent-S>

"**Agent S: an open agentic framework that uses computers like a
human.**" Agent-S (now `gui-agents` on PyPI, with Agent-S3 as the
current generation) is Simular's open framework for **GUI agents
that drive the desktop / web through screenshots + mouse +
keyboard**, not through DOM scraping or API calls. The agent
plans, observes pixels, grounds clicks via a separate visual-
grounding model, executes a `pyautogui` action, and re-observes —
the same loop OpenAI Operator and Anthropic Computer Use ship
behind closed APIs, runnable locally against your choice of
planner + grounder models.

## 1. Install footprint

- Library: `pip install gui-agents` exposes the `gui_agents`
  Python package; the CLI entrypoint `agent_s` runs an
  interactive desktop agent that screenshots the active display
  and emits actions.
- Required runtime deps: a planner LLM endpoint (OpenAI /
  Anthropic / Gemini / OpenRouter / Hugging Face / vLLM-served
  open weights), a visual-grounding endpoint (UI-TARS-1.5-7B
  served by vLLM / Hugging Face Inference, or Claude / GPT-4o
  reused as the grounder), plus screen-capture permissions on
  the host OS.
- Optional: `agent_s.cli_app --provider anthropic
  --grounding_model UI-TARS-1.5-7B` is the canonical desktop
  invocation; `OSWorld` benchmark scripts ship in `evaluation/`
  for reproducible scoring against the OSWorld task suite.

## 2. Repo + version + license

- Repo: <https://github.com/simular-ai/Agent-S>
- Latest release: **v0.3.2** (published 2025-12-16)
- License: **Apache-2.0** —
  <https://github.com/simular-ai/Agent-S/blob/main/LICENSE>
- Default branch: `main`
- Language: Python
- Stars: ~10.9k

## 3. Models supported

- Planner: OpenAI (GPT-4o / 4.1 / o-series), Anthropic (Claude
  3.5 / 3.7 / 4.x Sonnet), Gemini 2.x, OpenRouter aggregations,
  any OpenAI-compatible endpoint (vLLM / Together / Groq) for
  open-weight planners.
- Visual grounder (separate model): UI-TARS-1.5-7B (recommended,
  vLLM-served), Claude 3.7 / 4 Sonnet, GPT-4o, or any
  vision-capable model that returns pixel coordinates for a
  natural-language target description.
- Memory store: local JSON narrative + episodic memories per
  task; Agent-S3 introduces `Mixture-of-Grounding` to route
  ambiguous targets to multiple grounders and vote.

## 4. Notable angle

**Open desktop / web GUI agent with a pluggable planner + grounder
split.** Where [`browser-use`](../browser-use/),
[`stagehand`](../stagehand/), and
[`skyvern`](../skyvern/) automate inside the browser DOM (so they
can read accessibility-tree text and emit `click('#submit')`),
Agent-S goes one layer below: it screenshots the *whole desktop*,
asks a vision-capable planner for the next high-level action, and
hands off to a dedicated grounding model that returns the exact
pixel coordinates to click — the same architecture OpenAI Operator
and Anthropic Computer Use use, but with both halves swappable
between closed-frontier APIs and open weights (UI-TARS-1.5-7B
runs on a single 24 GB GPU). That makes Agent-S the only catalog
entry that can do "fill out this native macOS Settings pane" or
"drag this slide in PowerPoint-equivalent" without an OS-specific
accessibility-API integration. The trade-off: pixel-grounded
agents are slower per step than DOM-grounded ones, and the OSWorld
success rates remain well below human even at the SOTA. Use it
when DOM scraping is not an option (native apps, iframes,
Canvas-rendered UIs); use [`browser-use`](../browser-use/) or
[`stagehand`](../stagehand/) when you can stay in the browser.

## 5. Last verified

2026-04-26 via `gh api repos/simular-ai/Agent-S` and
`gh api repos/simular-ai/Agent-S/releases/latest` → v0.3.2.
