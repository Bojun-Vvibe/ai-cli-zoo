# promptfoo

> Prompt and LLM-output **evaluation** as a CLI: declarative test cases,
> matrixed providers, scriptable assertions, HTML report. As of v0.x
> series (late 2025).

## TL;DR

`promptfoo` is a test runner for prompts, in the same way `pytest` is a
test runner for Python. You write a `promptfooconfig.yaml` listing
prompts, providers (OpenAI, Anthropic, Gemini, Ollama, Bedrock,
Replicate, HuggingFace, custom HTTP, custom Python), and **test cases
with assertions** (`equals`, `contains`, `is-json`, `latency`, `cost`,
`llm-rubric`, `javascript`, `python`, `model-graded-closedqa`, etc.).
`promptfoo eval` runs the cartesian product, reports pass/fail per
cell, and emits an HTML diff view at `promptfoo view`. There is also
a built-in red-team mode (`promptfoo redteam`) that generates
adversarial prompts (jailbreak, PII leak, prompt injection patterns)
against your system prompt and grades responses.

## Install

```bash
# zero-install
npx promptfoo@latest eval

# global
npm install -g promptfoo
brew install promptfoo
```

Node 18+. No Python required. Provider keys via env (`OPENAI_API_KEY`,
`ANTHROPIC_API_KEY`, …) or per-provider `config.apiKey`.

## One Concrete Example

`promptfooconfig.yaml`:

```yaml
prompts:
  - "Summarize this in one sentence: {{text}}"
providers:
  - openai:gpt-4o-mini
  - anthropic:claude-3-5-haiku-latest
tests:
  - vars:
      text: "Promptfoo is a CLI for evaluating LLM prompts across providers."
    assert:
      - type: contains
        value: "evaluat"
      - type: llm-rubric
        value: "Response is one sentence and mentions Promptfoo."
      - type: latency
        threshold: 4000
```

Run:

```
$ promptfoo eval
┌─────────────────────────────────┬──────────────────────┬──────────────────────────────┐
│ text                            │ openai:gpt-4o-mini   │ anthropic:claude-3-5-haiku-… │
├─────────────────────────────────┼──────────────────────┼──────────────────────────────┤
│ Promptfoo is a CLI for eval…    │ ✓ PASS (842ms)       │ ✓ PASS (1.2s)                │
└─────────────────────────────────┴──────────────────────┴──────────────────────────────┘
2 of 2 tests passed.  Eval ID: eval-2025-…  →  promptfoo view
```

`promptfoo view` opens a local web UI with side-by-side diffs, pass/fail
filters, and per-assertion drill-down.

## Niche It Fills

**Reproducible prompt regression testing across providers.** Nothing
else in the catalog does this. Every other entry is for *producing*
output once; `promptfoo` is for *grading* output across many prompts ×
many providers × many test rows, and failing CI when a quality / cost
/ latency budget is broken. Drop it in a `pre-commit` or GH Actions
job and you get a real "did this prompt edit make the model worse"
signal instead of vibes.

## Vs Already Cataloged

- **Vs [`llm`](../llm/):** `llm` logs every prompt/response to SQLite
  for ad-hoc replay; `promptfoo` is the structured-eval layer. Use
  `llm` to *explore*, `promptfoo` to *gate*. They compose: `llm logs
  --json` → custom `promptfoo` provider that replays the row.
- **Vs [`fabric`](../fabric/):** `fabric` ships a curated library of
  prompts you *run*. `promptfoo` is the harness you point at those
  prompts to assert quality. A reasonable team workflow: author in
  `fabric`, regress in `promptfoo`.
- **Vs [`mods`](../mods/):** `mods` is a one-shot pipe; `promptfoo`
  runs the cartesian product. If you find yourself wrapping `mods` in
  a bash loop with `grep` assertions, that bash loop is `promptfoo`.

## Caveats

- `llm-rubric` and `model-graded-*` assertions cost real tokens — a
  100-row × 3-provider eval with rubric grading easily hits $1+.
  Pin `defaultTest.options.provider` to a cheap grader
  (`openai:gpt-4o-mini`) and watch `--max-concurrency`.
- Red-team mode generates adversarial inputs that look like real
  jailbreak attempts in CI logs; some org log scanners flag them.
  Run `redteam` in a separate, labeled job.
- The HTML viewer is local-only by default; `promptfoo share` uploads
  to a hosted bucket — opt-in, off by default, but worth knowing.
- Caching is on by default and keyed on `(prompt, provider, vars)`;
  if your provider returns different content for the same input
  (temperature > 0), set `--no-cache` or the eval becomes a tautology.
