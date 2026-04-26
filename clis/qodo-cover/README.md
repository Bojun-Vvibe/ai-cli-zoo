# qodo-cover

> Snapshot date: 2026-04-26. Upstream: <https://github.com/qodo-ai/qodo-cover>

"**AI-Powered Tool for Automated Test Generation and Code
Coverage Enhancement.**" Qodo-Cover (the OSS half of Qodo's
test-generation stack, formerly Codium-AI / Cover-Agent) is a
CLI that points at a source file + its existing test file +
the project's coverage report, and iteratively asks an LLM for
**new tests that strictly increase line/branch coverage and
actually pass**. Generated tests are run, parsed, and only kept
if they (a) compile, (b) pass, and (c) cover lines the existing
suite missed.

## 1. Install footprint

- Binary: `pip install qodo-cover` exposes the `cover-agent`
  console script (legacy entrypoint also kept for compatibility
  with older configs).
- Inputs: `--source-file-path`, `--test-file-path`,
  `--code-coverage-report-path` (Cobertura/JaCoCo/lcov), plus
  `--test-command` so the agent can rerun the suite after each
  proposed addition.
- Outputs: in-place edits to the test file, a markdown report of
  added tests + coverage delta, and a JSON log of every
  accepted/rejected candidate for audit.

## 2. Repo + version + license

- Repo: <https://github.com/qodo-ai/qodo-cover>
- Latest release: **v0.3.10**
- License: **AGPL-3.0** —
  <https://github.com/qodo-ai/qodo-cover/blob/main/LICENSE>
- Default branch: `main`
- Language: Python
- Stars: ~5.4k

## 3. Models supported

Backed by LiteLLM, so any OpenAI, Anthropic, Gemini, Mistral,
Groq, Bedrock, Vertex, or local Ollama/vLLM endpoint works via
`--model`. Default recipe targets a strong reasoning model for
test synthesis (GPT-4-class / Claude Sonnet) — weaker models
tend to produce tests that compile but don't actually exercise
the uncovered branch.

## 4. Notable angle

**Coverage report is the reward signal, not vibes.** Most "AI
writes your tests" tools stop at "here are some tests, hope
they're useful." Qodo-Cover treats test generation as a
**closed-loop optimization** against a real coverage tool: the
LLM proposes a test, the runner executes it, the coverage
delta is measured, and the test is only committed if coverage
strictly went up. The CI integration (`cover-agent-pr-action`,
GitLab CI recipe) makes this a per-PR gate — you can require
that any PR which drops coverage gets auto-augmented before
merge, with the human only reviewing the final diff.

## 5. Last verified

2026-04-26 via `gh api repos/qodo-ai/qodo-cover`.
