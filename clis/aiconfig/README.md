# aiconfig

> Snapshot date: 2026-04. Upstream: <https://github.com/lastmile-ai/aiconfig>

A **config-as-code framework for generative-AI prompts** —
prompts, model parameters, model identity, and chained prompt
graphs live in a JSON / YAML `aiconfig` file that ships with your
application as a versioned artefact, and the SDK
(`AIConfigRuntime` in Python / TypeScript) loads + runs that
config without prompt strings being hardcoded in source. The
goal: separate prompt engineering from application code so
non-engineers can iterate on prompts in a UI (`aiconfig edit`
opens a local notebook-style editor) while engineers ship
deterministic releases.

It is the catalog's reference for **prompt-as-portable-artefact
with a notebook-style local editor and SDK runtime in two
languages** — the Jupyter-inspired sibling to
[`pezzo`](../pezzo/)'s server-CMS approach.

## 1. Install footprint

- `pip install python-aiconfig` (Python 3.10+) or `npm install
  aiconfig` (Node 18+).
- Pulls `openai`, `pydantic`, `requests`. ~40 MB Python venv.
- Editor: `aiconfig edit my.aiconfig.json` boots a local Next.js
  notebook UI on `:8080` for prompt iteration.
- Models: OpenAI (default), Anthropic Claude, Google PaLM /
  Gemini, HuggingFace Inference, local via the model-parser
  extension surface (a model parser is a small Python class that
  knows how to invoke a given model family).

## 2. Repo, version, license

- Repo: <https://github.com/lastmile-ai/aiconfig>
- Version checked: **v1.1.32** (latest npm + PyPI tag).
- HEAD pinned at this snapshot:
  `cef0162b31b50dccbf42f1b0f5bb7627fb402e84`.
- License: MIT. License file at
  [`LICENSE`](https://github.com/lastmile-ai/aiconfig/blob/main/LICENSE).

## 3. What it actually does

The `aiconfig` JSON document is the source of truth. A minimal
example:

```json
{
  "name": "demo",
  "schema_version": "latest",
  "metadata": { "default_model": "gpt-4" },
  "prompts": [
    {
      "name": "summarize",
      "input": "Summarize this passage: {{passage}}",
      "metadata": { "model": "gpt-4", "parameters": { "passage": "" } }
    },
    {
      "name": "translate",
      "input": "Translate this summary to French: {{summarize.output}}",
      "metadata": { "model": "claude-3-opus" }
    }
  ]
}
```

In application code:

```python
from aiconfig import AIConfigRuntime, InferenceOptions
config = AIConfigRuntime.load("demo.aiconfig.json")
result = await config.run(
    "translate",
    params={"passage": open("essay.md").read()},
    options=InferenceOptions(stream=True),
)
```

The runtime resolves `{{summarize.output}}` by running
`summarize` first, threading the result into `translate`. Each
prompt's parameters, model id, and decoding settings live in the
config — change them in the editor, save, the running app picks
up the new version on the next `load()`.

The notebook-style editor renders each prompt as a cell with the
prompt text, the model dropdown, the parameters panel, the last
output, and a **Run** button. Non-engineers iterate on the prompt
without touching Python or TS.

## 4. MCP support

None first-party. The model-parser plugin surface is the
extension point — an `MCPModelParser` would be a small Python
class implementing the `ModelParser` interface, but no such
parser ships in v1.1.32.

## 5. Sub-agent model

None — aiconfig is a prompt-graph runtime, not an agent
framework. Sequential prompt chains are first-class (a prompt
references the output of an earlier prompt by name); branching
graphs (`if/else`, `parallel`) are not native primitives, you
chain at the application code level by calling `config.run(...)`
with conditional logic.

## 6. Telemetry stance

Off in OSS. The runtime emits structured callback events
(`on_prompt_start`, `on_prompt_end`, `on_token`) for application
code to wire to its own observability layer; LastMile AI's hosted
console is opt-in and reads emitted callbacks. Egress = whichever
model parsers you enable (each parser hits its provider's
endpoint).

## 7. Token / context strategy

Per-prompt: each prompt is its own model call with its own
parameters; no shared rolling context between prompts unless you
explicitly thread `{{previous.output}}` placeholders. This is the
opposite of an agent framework's accumulating context — aiconfig
treats each prompt as a stateless function call. Long-document
workloads (RAG over a 200-page PDF) are not the right shape for
aiconfig; pair with a real RAG framework
([`llama-index`](../llama-index/) / [`haystack`](../haystack/))
and use aiconfig only for the LLM-call hop.

## 8. Hot keybinds

The notebook editor (`aiconfig edit`) supports Jupyter-like
shortcuts: `Shift+Enter` runs the current prompt cell, `Ctrl+S`
saves the config to disk, `Ctrl+Z` undoes the last edit. The
shell CLI itself (`aiconfig run --config <file> --prompt <name>`)
is one-shot, no TUI.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Prompt-as-versioned-JSON-artefact** that
ships in the same Git repo as application code, with a local
notebook editor non-engineers can use. Prompt changes show up as
JSON diffs in PRs (legible code review for prompt updates),
prompt regressions roll back via `git revert`, and the
notebook editor closes the iteration loop without round-tripping
through engineering. The two-language SDK (Python and TypeScript
load the same JSON file with identical semantics) is rare in this
space.

**Weakness.** Smaller community than [`pezzo`](../pezzo/) /
[`langfuse`](../langfuse/) / [`promptfoo`](../promptfoo/);
release cadence has slowed since v1.1.x stabilized. Model-parser
extension surface means exotic providers (local llama.cpp,
Bedrock, Vertex) require writing your own parser if not in the
shipped registry. No built-in eval / regression harness — pair
with [`promptfoo`](../promptfoo/) or [`deepeval`](../deepeval/)
for "did this prompt edit regress my golden set" checks. The
prompt-graph model is intentionally simple — agentic behavior
(tool calls, conditional branching, retries on validation
failure) lives in your application code, not in the config.

**Choose aiconfig when** prompts need to ship as versioned
artefacts in the application repo, non-engineers must own prompt
iteration without a server-side CMS, and the same prompt graph
runs identically from Python services and TypeScript edge
functions. **Choose something else when** you need a hosted
prompt CMS with environment-label deploys (use
[`pezzo`](../pezzo/)), when bundled traces + evals + datasets in
one tool matter (use [`langfuse`](../langfuse/) /
[`laminar`](../laminar/)), or when the workload is a real
multi-step agent loop with tool calls (use
[`pydantic-ai`](../pydantic-ai/) /
[`openai-agents-python`](../openai-agents-python/)).
