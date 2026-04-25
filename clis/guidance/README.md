# guidance

> Snapshot date: 2026-04. Upstream: <https://github.com/guidance-ai/guidance>
> License file: <https://github.com/guidance-ai/guidance/blob/main/LICENSE.md>
> Pinned: `0.3.2` (2026-03-18, Python package). Default branch is
> `main`. HEAD verified at `5413339aad8d36ce49df29902a7730d025bd6027`.
> Install as `pip install guidance`.

A **constrained-generation language for LLMs** — `guidance` lets you
interleave prompt strings, sampler constraints (`select`, `gen`,
`regex`, `json`), control flow (`if`, `for`), and tool calls in one
Python program that the runtime executes against a local or remote
model token-by-token, masking the logits so invalid tokens never
sample. The contrast against [`outlines`](../outlines/) is "imperative
DSL with token-stream control flow" rather than "declarative regex /
JSON schema → finite-state automaton over the sampler", and against
[`instructor`](../instructor/) is "constrained at decode time, output
is structurally guaranteed" rather than "validate after the fact +
re-ask on parse failure".

## 1. Install footprint

- **Library**: `pip install guidance` (~50 MB with deps —
  `pydantic`, `numpy`, `tiktoken`, `protobuf`, `referencing`,
  `jsonschema`, `numba`, `llguidance`).
- **Local-model extras**: `pip install guidance[transformers]`
  pulls `torch` + `transformers` for `models.Transformers(...)`;
  `guidance[llamacpp]` adds `llama-cpp-python` for GGUF; native
  `models.LlamaCpp(...)`, `models.LiteLLM(...)`, `models.OpenAI(...)`,
  `models.Anthropic(...)`, `models.Azure*(...)`, `models.VertexAI(...)`
  bindings.
- **No standalone CLI** — the surface is a Python module; you
  embed `guidance` in an existing script, notebook, or service. The
  Rust core (`llguidance`) ships compiled wheels for x86\_64 + arm64
  Linux / macOS / Windows; no Rust toolchain needed at install.

## 2. License

MIT (file is `LICENSE.md` at the repo root,
<https://github.com/guidance-ai/guidance/blob/main/LICENSE.md>).
The project moved from the original incubator to the `guidance-ai`
GitHub org and remains permissively licensed; `llguidance` (the Rust
sampler core that powers vLLM / SGLang / TGI server-side constrained
decoding) is also MIT.

## 3. Models supported

Two operating modes:

- **Full token-mask control** — local models loaded via
  `transformers` (any HF causal LM), `llama-cpp-python` (GGUF), or a
  vLLM / SGLang / TGI server with `llguidance` enabled. Here
  guidance manipulates per-token logit masks, so `select(["A", "B"])`,
  `gen(regex=r"\d{4}")`, and `json(schema=MySchema)` produce output
  that is **structurally impossible to violate**.
- **Constrained-prompt fallback** — remote chat-completion APIs
  (OpenAI, Anthropic, Gemini via LiteLLM) where token logits are
  not exposed; guidance falls back to JSON-mode / tool-calling /
  re-ask, similar to [`instructor`](../instructor/). Local mode is
  the design centre — that's where the value is.

`llguidance`-aware servers (vLLM ≥ 0.7, SGLang, TGI) accept the
guidance grammar over HTTP, so the local-mode guarantees extend to
server-deployed models without bundling a sampler in every client.

## 4. MCP support

No first-party MCP server. Guidance is a sampler-level decoding
library, not a tool registry — the obvious wiring is "wrap a
guidance program as an MCP tool" via the reference `mcp.server` SDK,
exposing the typed output as a tool result. A guidance program *can*
call external tools mid-generation via `@guidance(stateless=False)`
functions that pause the sampler, run arbitrary Python (an MCP
client call, an HTTP request, a search), and resume — but routing
the *agent's* tool calls through MCP is the host's job, not
guidance's.

## 5. Sub-agent model

Single-program model. The `lm += "..."` operator threads an LM state
through the program; `@guidance` decorators define reusable
sub-routines (think "function with prompt + sampler context").
There's no scheduler / supervisor — for multi-agent orchestration
pair with [`langgraph`](../langgraph/), [`autogen`](https://github.com/microsoft/autogen)-style
runtimes, or call multiple guidance programs from a parent agent
loop. The mental model is "guidance is what you call when you'd
otherwise hand-code a regex parser around `chat.completions.create`".

## 6. Telemetry stance

Off. The library does not phone home; remote-model calls go to
whatever endpoint you configure (`OpenAI(api_key=...)` etc.). Local-
model loads pull weights from HuggingFace Hub on first use unless
you pre-stage them. No analytics, no anonymous metrics — for
tracing wire OpenInference / OpenTelemetry around the guidance call
site (`opentelemetry-instrumentation-openai` covers the OpenAI path;
local-model paths require manual span creation around `lm += gen(...)`).

## 7. Prompt-cache strategy

**KV-cache reuse is the killer optimization** — when guidance's
control flow branches (`if`/`select`/`for` over a partial completion),
the runtime keeps the prefix's transformer KV cache and only re-
processes from the divergence point. Equivalent hand-coded "ask the
model A, then ask the model B" against an HTTP API would re-encode
the prompt twice; in local mode guidance encodes once. For remote
APIs this advantage is unavailable (the provider does not expose
cache state); pair with Anthropic / Google native prompt caching
when remote-mode is required.

## 8. Hot keybinds

No TUI. The surface is a Python module:

```python
import guidance
from guidance import models, gen, select, json as gjson
from pydantic import BaseModel

llm = models.LlamaCpp("./Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf",
                     n_ctx=4096, n_gpu_layers=-1)

class Triage(BaseModel):
    severity: str  # constrained below
    next_action: str
    eta_hours: int

prompt = (
    "Classify this incident:\n"
    "Title: db_replica lag spike to 90s\n"
    "Body: replica-3 in us-east-1 fell behind primary at 14:02 UTC\n\n"
    "severity: " + select(["S0", "S1", "S2", "S3"], name="sev")
    + "\nnext_action: " + gen(name="action", stop="\n", max_tokens=50)
    + "\neta_hours: " + gen(name="eta", regex=r"\d{1,3}")
)

out = llm + prompt
print(out["sev"], out["eta"], out["action"])  # all structurally valid
```

For full-schema constrained JSON: `lm += gjson("incident",
schema=Triage)` emits exactly one Triage-shaped object with no
post-hoc parsing.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Token-level constrained decoding that makes
invalid output structurally impossible**, plus **KV-cache reuse
across control-flow branches** — `select(["yes", "no"])` is one
forward pass that masks every other token, not "ask the model and
hope it answers yes/no". The same grammar runs inside vLLM / SGLang
/ TGI via `llguidance` so the guarantee extends to server-deployed
fleets. For pipelines where "the JSON is malformed" is a costly
failure mode (function calling, agent tool dispatch, structured ETL),
guidance closes that class of bug at the sampler.

**Weakness.** The benefits collapse against remote-only providers
(OpenAI / Anthropic / Gemini chat completions) that don't expose
logits — there guidance's value reduces to a typed wrapper similar
to [`instructor`](../instructor/) / [`outlines`](../outlines/). The
DSL has a learning curve (`lm += "..."` operator overloading,
`@guidance(stateless=...)` semantics, `select` vs `gen` vs `json`)
and the v0.3 rewrite shifted internals significantly — pre-0.2
recipes do not run. Heavy lift if all you want is "LLM returns a
Pydantic model"; choose [`instructor`](../instructor/) for that.

**When to choose.**

- You run **local models** ([`vllm`](../vllm/), [`llamafile`](../llamafile/),
  llama.cpp, HF transformers) and need **structurally guaranteed
  output**.
- Your pipeline branches on partial completions and you want
  **KV-cache reuse** across branches (multi-step extraction, chain-
  of-thought with structured intermediate outputs).
- You ship a **JSON-schema-shaped agent tool call** and "model
  returned malformed JSON" is unacceptable — pair with
  `llguidance`-enabled vLLM in prod.

**When not to choose.** You only call hosted chat-completion APIs —
the constrained-decoding advantage is unavailable; pick
[`instructor`](../instructor/) (lighter) or [`outlines`](../outlines/)
(declarative). You need orchestration / multi-agent / tool-routing —
guidance is a decoder, not a runtime; pair with
[`langgraph`](../langgraph/) / [`agno`](../agno/) /
[`pydantic-ai`](../pydantic-ai/). You want a no-code authoring shell
— [`langflow`](../langflow/) is the canvas analogue.
