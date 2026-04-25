# baml

> Snapshot date: 2026-04. Upstream: <https://github.com/BoundaryML/baml>

**A typed DSL + compiler for prompts, with a real CLI.** BAML
(Boundary AI Markup Language) is a `.baml` source file you write
alongside your application code that declares LLM "functions" with
typed inputs, typed outputs, the prompt template, the model, and
test cases. The `baml-cli` then code-generates idiomatic client
libraries for Python / TypeScript / Ruby / Java / C# / Rust / Go,
runs the test suite (`baml-cli test`), and ships a VS Code
playground (`baml-cli dev`) with prompt diffing + structured-output
preview against live providers.

## Repo + version + license

- Repo: <https://github.com/BoundaryML/baml>
- Latest release: **`0.221.0`** (2026-04-24)
- HEAD on `canary`: `786aa50`
- License: **Apache-2.0** —
  <https://github.com/BoundaryML/baml/blob/canary/LICENSE>
- License path in repo: `LICENSE`
- License SPDX: `Apache-2.0`
- Default branch: `canary`
- Language: Rust (compiler) + generated clients in 7 host languages

## Install

```bash
# Per-host-language install (pulls the prebuilt CLI as a wheel/binary)
pip install baml-py            # Python
npm install -g @boundaryml/baml # TypeScript / Node

# Then in your repo:
baml-cli init                  # scaffold baml_src/ + clients.baml
baml-cli generate              # emit typed client into your codebase
baml-cli test                  # run all `test` blocks against real providers
baml-cli dev                   # boot VS Code playground server
```

## Niche

The "**typed prompts that compile to client code**" lane.
Adjacent to [`instructor`](../instructor/),
[`outlines`](../outlines/),
[`marvin`](../marvin/), [`mirascope`](../mirascope/), and
[`pydantic-ai`](../pydantic-ai/) on the structured-output axis,
but inverts the relationship: the prompt and its types live in a
neutral `.baml` file, and Python / TS / Ruby / Java / C# / Rust /
Go all import the *same* generated client — useful for polyglot
teams where one prompt powers a Python backend, a TS frontend, and
a Ruby on Rails admin tool without three drifting copies.

## Why it matters

- **The DSL is the contract.** A `function ExtractInvoice(pdf:
  string) -> Invoice { client GPT5 prompt #"..."# }` block declares
  the signature, the model, and the prompt in one place; the
  generated `await b.ExtractInvoice(pdf)` call returns a typed
  `Invoice` object with parsed fields, retries, and structured-
  error handling — no Pydantic dance, no JSON-mode wrangling, no
  per-host-language re-implementation of the same coercion logic.
- **`baml-cli test` is real testing.** Each `.baml` file holds
  `test` blocks with named inputs and assertions; `baml-cli test`
  fans out across providers + models + temperature in parallel,
  produces a pass/fail matrix, and writes JUnit XML for CI. Prompt
  regressions get caught before merge.
- **VS Code playground (`baml-cli dev`).** Hot-reload on `.baml`
  save, side-by-side diff of prompt versions, structured-output
  preview against live providers, token-cost meter per call, and a
  one-click "promote this prompt to the main file" flow.
- **Polyglot by design.** One `baml_src/` directory, seven
  generated clients, one source of truth for the prompt + the type
  + the model selection — kills the "Python team and TS team have
  silently diverged on the system prompt" failure mode.
- **OpenAI-shaped, not OpenAI-locked.** `clients.baml` declares the
  provider matrix (Anthropic, OpenAI, Gemini, Bedrock, Azure,
  Ollama, vLLM, OpenRouter, Together, Groq, any OpenAI-compatible
  URL); switching the model behind a function is a one-line edit
  to the `client` annotation, not a rewrite of the call site.
