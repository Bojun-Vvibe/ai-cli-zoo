# lmql

> Snapshot date: 2026-04. Upstream: <https://github.com/eth-sri/lmql>
> Pinned commit: `fc8edd4457dc5a5e4e595302166bec797582bf53` (tag `v0.7.3`).
> License: Apache-2.0 — `LICENSE` at repo root.

A **constraint-guided query language** for LLMs from ETH Zurich, shipped as a
Python package with three CLIs (`lmql run`, `lmql playground`, `lmql serve-model`)
and an in-browser playground. It sits in an unusual catalog slot: not a
"chat with a model" CLI like `aichat`, not an autonomous agent like `aider`,
but a **declarative DSL** where you write Python-like programs whose
control-flow is decoded tokens, with `where` clauses that constrain
generation at the logits level (regex, length, type, custom predicates).

The right pick when you want **structured generation with hard guarantees**
(JSON shape, integer range, vocabulary subset) without writing a JSON-schema
prompt and praying the model honors it.

## 1. Install footprint

- `pip install lmql` (Python 3.10+). Optional extras: `lmql[hf]` for local
  HuggingFace transformers backend, `lmql[replicate]`, `lmql[llama]` for
  llama.cpp.
- No daemon. The `lmql serve-model` CLI is opt-in: it stands up a local
  HTTP server that hosts a single HF / llama.cpp model and exposes it to
  `lmql run` programs over the same machine — useful for keeping a 7B
  model resident across many query runs.
- Three CLI entry points installed by the package:
  - `lmql run script.lmql` — execute a `.lmql` query file
  - `lmql playground` — open the local playground (browser UI on `:3000`)
  - `lmql serve-model <hf-name>` — host a model for `lmql run` clients
- Project-local `.lmql` files; no global state directory required.

## 2. License

Apache-2.0. `LICENSE` is at the repo root, SPDX-detectable.

## 3. Models supported

- **OpenAI** Chat + completion endpoints (`openai/gpt-4o`, `openai/gpt-3.5-turbo`,
  etc.).
- **Anthropic** via the `anthropic/` prefix (Claude 3 family).
- **HuggingFace transformers** — any causal-LM checkpoint, loaded locally
  (`local:meta-llama/Llama-3.1-8B-Instruct`).
- **llama.cpp** GGUF models via the `llama.cpp:` prefix.
- **Replicate** hosted models.
- Azure OpenAI deployments by setting `OPENAI_API_TYPE=azure` plus the
  endpoint env vars.

The constraint engine works on **all** of these, but the *strength* of
the constraints differs: for `local:` and `llama.cpp:` backends LMQL has
direct logits access and can mask token-by-token; for OpenAI / Anthropic
chat models it falls back to a **rejection-sampling** loop (generate a
token, check, retry) which is correct but slower and burns more API
budget on tight constraints.

## 4. MCP support

None. LMQL predates the MCP spec and the maintainers' focus shifted to
research before MCP became standard. There is no `lmql --mcp` server, no
client. If you need MCP tool-calling from a structured-generation DSL,
this is not the pick — `instructor` + `mcp-use`, or `baml`, are closer.

## 5. Sub-agent model

None. LMQL is a single-program execution model: one `.lmql` file, one
forward pass through the query, one set of variables bound. You can
*compose* queries by calling one `.lmql` from another via Python imports,
but they execute serially in the same process — no planner/builder split,
no parallel workers.

## 6. Telemetry stance

**Off, no opt-in.** No analytics in the package. Outbound network is
exclusively to the model backend you configured (OpenAI / Anthropic /
Replicate / your own `serve-model`). The playground is fully local;
no phone-home on `lmql playground`.

## 7. Prompt-cache strategy

Per-backend pass-through. For `local:` HF models LMQL keeps the
transformers KV-cache alive across `where`-clause rejection-and-resample
cycles in a single query, which is the dominant speedup on constrained
generation (you don't re-prefill the prompt for every rejected token).
For OpenAI / Anthropic backends there is no cross-call cache at the LMQL
layer — Anthropic prompt caching applies if your account has it on, but
LMQL does not set the `cache_control` headers for you in v0.7.

## 8. Hot keybinds

LMQL is mostly file-driven, not REPL-driven. The CLIs:

- `lmql run script.lmql` — runs the query, prints variable bindings.
  Flags worth knowing: `--output json` (machine-readable), `--decoder
  beam_search`, `--decoder sample`, `--n 4` (multiple completions).
- `lmql run --interactive script.lmql` — pause after each `decoder`
  step so you can inspect partial state.
- `lmql playground` — opens browser UI; in-browser keybinds: `Ctrl/Cmd-Enter`
  runs the current query, `Ctrl/Cmd-S` saves to local storage,
  `Ctrl/Cmd-K` opens the model picker.
- `lmql serve-model meta-llama/Llama-3.1-8B-Instruct --cuda --port 8080`
  — host a model with KV-cache reuse across clients.

The DSL itself: `argmax "Q: [QUESTION] A: [ANSWER]" where len(TOKENS(ANSWER)) < 100`
is the canonical shape. `where` accepts `INT(x)`, `STOPS_AT(x, ".")`,
`x in set(["yes","no"])`, regex, and Python predicates.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Logits-level constraints with declarative syntax.**
For local HF / llama.cpp models, `where x in ["yes","no","maybe"]`
literally masks the logits at every decoding step so the model
**cannot** emit anything else — no rejection loop, no JSON-schema retry,
no "the model added trailing prose". This is the same primitive `outlines`
and `guidance` provide, but LMQL exposes it through a Python-superset
language that reads top-to-bottom like a normal program, not as a chain
of object-mode API calls. For research workflows that need "generate
with these guarantees and tell me the per-token logprob" it remains the
most ergonomic choice.

**Weakness.** Maintenance has slowed: last tagged release `v0.7.3` was
2024-04, last commit on `main` 2025-05. Newer model families (GPT-5,
Claude 3.7, o-series) work via the OpenAI / Anthropic backends but the
**rejection-sampling** path on those hosted models can be expensive for
tight constraints. Tooling around `.lmql` files (LSP, formatter) is
thin. The DSL has its own grammar — not Python, not Jinja — which is
another thing to learn for one workflow.

**When to choose.**

- You need **token-level constraints** (regex, type, vocabulary subset,
  length bound) on a **local** HF or llama.cpp model and want speed plus
  guarantees, not a JSON-schema retry loop.
- You write **research-grade prompting experiments** where reproducibility
  matters: the same `.lmql` file with a fixed seed and a fixed model is
  deterministic, including the constraint-rejection path.
- You want a **single declarative artifact** (the `.lmql` file) instead of
  a Python script that calls a model client and a separate JSON-schema
  validator.

**When not to choose.** You want autonomous file edits (use `aider`,
`codex`), you want MCP tools (`goose`, `opencode`), you want structured
output as a thin Python decorator (`instructor`, `outlines`), you target
hosted-only chat models where the rejection-sampling path negates the
constraint-speedup story (use `instructor` + Pydantic), or you need a
project on active 2026 release cadence.
