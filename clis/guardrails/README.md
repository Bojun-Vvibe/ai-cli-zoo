# guardrails

> Snapshot date: 2026-04. Upstream: <https://github.com/guardrails-ai/guardrails>
> License file: <https://github.com/guardrails-ai/guardrails/blob/main/LICENSE>
> Pinned: `v0.10.0` (2026-04-03, Python package). Default branch is
> `main`. Install as `pip install guardrails-ai` (the PyPI name diverges
> from the import name `guardrails`).

A **declarative validation + structured-output framework** for LLM
calls. You author a `RAIL` spec (XML) or a Pydantic model, wrap any
chat-completion call in a `Guard`, and the library enforces typed
output, range / regex / semantic constraints, and re-asks the model
on failure. The contrast point against [`instructor`](../instructor/)
is "validation as a first-class layer with a Hub of pluggable
validators" rather than "Pydantic-shaped retry decorator" — Guardrails
ships a registry (`hub.guardrailsai.com`) of ~60 prewritten checks
(PII redaction, profanity, competitor mentions, JSON schema, SQL
injection, prompt-injection heuristics, toxicity).

## 1. Install footprint

- **Library**: `pip install guardrails-ai` (~30 MB with deps:
  `pydantic`, `lxml`, `litellm`, `tenacity`).
- **Hub validators** (opt-in, downloaded on demand):
  `guardrails hub install hub://guardrails/detect_pii` — each
  validator is its own pip package under `guardrails-grhub-*`.
- **Server mode**: `guardrails start --config config.py` exposes an
  OpenAI-compatible `/v1/chat/completions` proxy that applies the
  Guard server-side, so polyglot clients can opt in without per-
  language SDKs.
- **CLI**: `guardrails configure` (set Hub token), `guardrails hub
  list / install / uninstall`, `guardrails validate <rail.xml>`.

## 2. License

Apache-2.0 (file is `LICENSE` at the repo root,
<https://github.com/guardrails-ai/guardrails/blob/main/LICENSE>).
The core library and the published Hub validators are Apache-2.0;
Guardrails Pro (managed Hub + telemetry dashboard) is the commercial
offering and is out of band.

## 3. Models supported

Provider-agnostic — under the hood `Guard.__call__` dispatches to
`litellm`, so anything `litellm` can reach (OpenAI, Anthropic, Google,
Mistral, Cohere, Bedrock, Azure-OpenAI-shaped endpoints, Ollama,
vLLM, llama.cpp servers) works. For "validator runs a smaller model
to grade the big model's output" patterns (toxicity, hallucination,
NLI-grounded citation), the Hub validators pin their own
HuggingFace models and download on first use.

## 4. MCP support

No first-party MCP server. The Guard surface is a Python decorator
+ HTTP server, not a tool registry — the obvious wiring is "expose
Guard-protected functions as MCP tools" via a generic
Python-MCP bridge ([`fastmcp`](https://github.com/jlowin/fastmcp)
or `mcp.server` from the reference SDK). The server mode's OpenAI-
compatible endpoint is the easier integration path: any MCP-aware
agent that already speaks OpenAI chat completions ([`opencode`](../opencode/),
[`claude-code`](../claude-code/), [`crush`](../crush/)) can route
through `guardrails start` to get validation for free.

## 5. Sub-agent model

The Guard itself is not multi-agent, but the **re-ask loop** is the
sub-agent shape: when a validator fails, the library generates a
follow-up prompt that quotes the failed output + the validator's
error and sends it back to the same model (configurable: `reask`,
`fix`, `noop`, `exception`, `filter`). Custom validators are plain
Python classes with `validate(value, metadata) -> ValidationResult`,
so a "second model grades the first" sub-agent is straightforward.

## 6. Telemetry stance

Anonymous CLI telemetry on by default; disable with
`guardrails configure --disable-metrics-reporting` or env var
`GUARDRAILS_DISABLE_METRICS=true`. Hub installs phone home to the
Hub registry to resolve packages — for fully air-gapped use, mirror
the validator wheels and pin them in `requirements.txt` instead of
using `guardrails hub install`. The library itself does not log
prompts or completions; OpenInference / OpenTelemetry tracing is
opt-in via `guardrails.telemetry.default_otel_collector`.

## 7. Prompt-cache strategy

None at the framework layer. Caching belongs to the underlying
provider call — pair with [`gptcache`](../gptcache/) on the LiteLLM
side, or use Anthropic / Google native prompt caching by passing
provider-specific `extra_body` through the Guard. The re-ask loop
does *not* cache failed outputs, which is intentional: the second
ask needs to see a fresh completion or it will loop forever on the
same wrong answer.

## 8. Hot keybinds

No TUI. The everyday surface is a Python import.

```python
from pydantic import BaseModel, Field
from guardrails import Guard
from guardrails.hub import DetectPII, ToxicLanguage

class Reply(BaseModel):
    summary: str = Field(..., description="1-2 sentences, no PII")
    confidence: float = Field(ge=0.0, le=1.0)

guard = Guard.for_pydantic(Reply).use_many(
    DetectPII(pii_entities=["EMAIL_ADDRESS", "PHONE_NUMBER"], on_fail="fix"),
    ToxicLanguage(threshold=0.5, on_fail="exception"),
)

result = guard(
    model="openai/gpt-4o-mini",
    messages=[{"role": "user", "content": "Summarise this ticket: ..."}],
)
print(result.validated_output)  # typed dict matching Reply
```

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A pluggable Hub of ~60 prewritten validators**
(PII, profanity, competitor mentions, JSON schema, SQL safety,
prompt-injection heuristics, hallucination grounding, NLI citation
checks) that compose into a single `Guard` and run as a re-ask loop
against any LiteLLM-reachable provider. The server mode means a
polyglot stack can opt in by changing one base URL — no per-
language Pydantic port required.

**Weakness.** The XML-flavoured `RAIL` spec is the historical
authoring shape and feels heavy next to a plain Pydantic model;
the Hub install workflow (`guardrails hub install`) phones home and
pulls per-validator wheels at runtime, which complicates air-gapped
deployments. The framework is heavier than [`instructor`](../instructor/)
or [`outlines`](../outlines/) — if all you want is "Pydantic out,
retry on parse failure", those are leaner.

**When to choose.**

- You need **content-safety validators** (PII, toxicity, prompt-
  injection) on top of structured output, and you don't want to
  hand-roll them.
- You want a **server-side validation proxy** so a polyglot stack
  (Node, Go, Rust) gets the same Guard without per-language SDKs.
- You need **declarative re-ask semantics** (`fix` / `reask` /
  `exception` / `filter`) configured per-validator.

**When not to choose.** You only need typed output from one provider
— [`instructor`](../instructor/) is the lighter choice. You want
*grammar-constrained* decoding that makes invalid output structurally
impossible — [`outlines`](../outlines/) compiles a regex / JSON
schema into the sampler instead of validating after the fact. You
want eval-shaped grading rather than per-call gating —
[`deepeval`](../deepeval/) or [`ragas`](../ragas/) is the right
layer.
