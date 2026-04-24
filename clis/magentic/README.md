# magentic

> Snapshot date: 2026-04. Upstream: <https://github.com/jackmpcollins/magentic>

A Python library + thin CLI that frames LLM calls as **type-annotated
Python functions**. You write a function signature with a docstring,
decorate it with `@prompt` or `@chatprompt`, and `magentic` turns the
return-type annotation into a structured-output contract enforced
against the model.

It earns its slot in the catalog because it is the cleanest CLI/SDK in
the zoo for the *function-calling-as-API* pattern: no agent loop, no
tool registry, no TUI — just `def(...)-> MyPydanticModel:` and the
library does the rest.

## 1. Install footprint

- `pip install magentic` (or `uv pip install magentic`).
- Pure-Python; pulls in `pydantic`, `openai`, and optional extras for
  Anthropic / LiteLLM (`pip install 'magentic[anthropic]'`,
  `pip install 'magentic[litellm]'`).
- No daemon, no project-local state. Configuration is environment
  variables (`MAGENTIC_BACKEND`, `MAGENTIC_OPENAI_MODEL`, etc.) or a
  per-call `OpenaiChatModel(...)`/`AnthropicChatModel(...)` argument.
- The `magentic` shell entry-point is a thin wrapper for running a
  `.py` file that contains decorated functions and printing their
  return values; the bulk of usage is `python -m` or in-process import.

## 2. License

MIT.

## 3. Models supported

- OpenAI (built-in): GPT-4.x, o-series, omni, any chat-completions
  endpoint.
- Anthropic (built-in extra): Claude 3 / 3.5 / 4 family with native
  tool-use protocol.
- LiteLLM extra: routes to ~100 providers (Gemini, Bedrock, Vertex,
  Mistral, Groq, Cohere, Ollama, vLLM, OpenRouter, Together, Fireworks,
  DeepSeek, Qwen) — but you lose native streaming-of-structured-output
  on backends that don't support function calling natively.
- Backend is selected per call (`OpenaiChatModel("gpt-4o")`,
  `AnthropicChatModel("claude-sonnet-4")`) or globally via
  `MAGENTIC_BACKEND` + `MAGENTIC_*_MODEL`.

## 4. MCP support

**No.** `magentic` does not consume or expose MCP servers. Tools are
plain Python callables registered via the `functions=` argument or the
`@chatprompt(functions=[...])` decorator, dispatched through native
provider function-calling.

## 5. Sub-agent model

**None.** There is no agent loop and no sub-agent concept. A `@prompt`
function is a single round-trip; a `@chatprompt` function with
`functions=[...]` is a single round-trip per tool call, and *you* drive
the loop in Python (typically a `while not done:` over the returned
`FunctionCall`). This is a deliberate design choice — the library
stays out of the orchestration layer.

## 6. Telemetry stance

**Off.** No analytics, no phone-home. Network egress is exclusively
to the model provider you configured. Logging is opt-in via
`logging.getLogger("magentic")` and an optional `litellm.success_callback`
hook for those using the LiteLLM backend.

## 7. Prompt-cache strategy

None at the library layer. `magentic` passes prompts through verbatim
so any prefix-cache behavior is whatever the underlying provider
offers (Anthropic prompt caching, OpenAI automatic prefix caching,
Gemini context caching). The library does *not* re-order or rewrite
the system / user blocks, which keeps provider-side caches valid
between calls.

## 8. Hot keybinds

N/A — there is no TUI or REPL. Usage is `python script.py` or import
into a notebook.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Type-annotated structured output as the *primary*
API. Annotate a function `-> list[Recipe]` where `Recipe` is a Pydantic
model and the library generates the JSON-schema function call,
validates the response, and returns native Python objects — all
streaming-aware (`StreamedResponse[Recipe]` yields completed objects
as they parse). Compared to writing OpenAI function-calling glue by
hand, this collapses ~30 lines of boilerplate per call into a single
decorated signature. The closest neighbor in the zoo is
[`chatblade`](../chatblade/) (`-e '.path' -j` extracts a JSON sub-path
from a one-shot CLI call), but `chatblade` is a shell tool with
JSONPath; `magentic` is a library with full Pydantic validation,
streaming, and provider-native function calling.

**Weakness.** It is a library first and a CLI second. If you wanted a
TUI, an agent loop, file-editing tools, or anything that walks a
codebase, you are in the wrong entry — pick [`aider`](../aider/),
[`opencode`](../opencode/), or [`claude-code`](../claude-code/). Also,
`magentic` is opinionated about Pydantic v2 — older codebases on
v1 cannot adopt it without migration.

**When to choose.**

- You are building a Python service / batch job / notebook and want
  LLM calls to feel like ordinary typed function calls.
- You need streaming of *structured* output (objects materialize as
  they parse, not after the full response arrives).
- You want one syntax that works against OpenAI native function
  calling and Anthropic native tool use without rewriting the call
  sites.

**When NOT to choose.**

- You want an interactive coding assistant (use `aider`, `opencode`,
  `claude-code`, `cline`).
- You want a shell pipe (use [`mods`](../mods/), [`smartcat`](../smartcat/),
  [`llm`](../llm/)).
- You want an agent loop that drives tools to completion on its own
  (use [`gptme`](../gptme/), [`open-interpreter`](../open-interpreter/),
  [`OpenHands`](../openhands/)).
- You are not on Pydantic v2 and cannot migrate.

## Representative invocation

```python
# recipes.py
from magentic import prompt
from pydantic import BaseModel

class Recipe(BaseModel):
    name: str
    ingredients: list[str]
    steps: list[str]

@prompt("Give me three recipes that use {ingredient}.")
def suggest_recipes(ingredient: str) -> list[Recipe]: ...

if __name__ == "__main__":
    for r in suggest_recipes("eggplant"):
        print(r.model_dump_json(indent=2))
```

```bash
$ MAGENTIC_BACKEND=openai MAGENTIC_OPENAI_MODEL=gpt-4o python recipes.py
{
  "name": "Roasted Eggplant with Tahini",
  "ingredients": ["1 eggplant", "2 tbsp tahini", ...],
  "steps": ["Preheat oven to 220C", ...]
}
...
```
