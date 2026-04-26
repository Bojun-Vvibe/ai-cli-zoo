# lavague

> Snapshot date: 2026-04-26. Upstream: <https://github.com/lavague-ai/LaVague>

"**Large Action Model framework to develop AI Web Agents.**"
LaVague is an open-source framework that turns natural-language
objectives into browser actions via a two-step World Model →
Action Engine pipeline: a planner LLM reads the rendered DOM +
goal and emits a high-level instruction, then a smaller code-gen
LLM compiles that into a Selenium/Playwright snippet which
LaVague executes against a real browser session.

## 1. Install footprint

- Library: `pip install lavague` exposes the `lavague` Python
  package; sub-packages `lavague-core`, `lavague-drivers-selenium`,
  `lavague-drivers-playwright` split planner from executor.
- CLI: `lavague` entrypoint with `launch`, `build`, and `exec`
  subcommands for spinning up an agent against a URL, generating
  reusable scraper code, and replaying recorded trajectories.
- Default driver is Selenium + Chrome; Playwright driver is
  optional. Models are LiteLLM-routed so any OpenAI/Anthropic/
  Gemini/Mistral/local backend works.

## 2. Repo + version + license

- Repo: <https://github.com/lavague-ai/LaVague>
- Latest tag: **v1.0.22**
- License: **Apache-2.0** —
  <https://github.com/lavague-ai/LaVague/blob/main/LICENSE>
- Default branch: `main`
- Language: Python
- Stars: ~6.3k

## 3. Models supported

Planner (World Model) and executor (Action Engine) are picked
independently via LiteLLM strings, so any combination of
OpenAI GPT-4-class, Anthropic Claude, Gemini, Mistral Large,
Groq, or a locally-served Llama/Qwen via Ollama/vLLM is valid.
The default recipe pairs a strong planner with a cheap code-gen
model to keep per-step cost bounded.

## 4. Notable angle

**Web agent that compiles to replayable Selenium/Playwright code,
not just live clicks.** The Action Engine emits actual Python
snippets, so a successful run leaves you with a deterministic
scraper you can commit, schedule, and diff — not an opaque agent
trace. LaVague Build mode is explicit about this: feed it a goal,
get back a `.py` file you can lint and rerun without an LLM in
the loop. That makes it a practical bridge between "I want an
agent to fill this form" prototyping and "ship a maintained
automation" production.

## 5. Last verified

2026-04-26 via `gh api repos/lavague-ai/LaVague`.
