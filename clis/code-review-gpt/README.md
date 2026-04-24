# code-review-gpt

> Snapshot date: 2026-04. Upstream: <https://github.com/mattzcarey/code-review-gpt>
> Binary name: `code-review-gpt`

`code-review-gpt` is the **diff-as-input code reviewer** of the catalog.
Where every other entry in the zoo *produces* code or *transforms* a
prompt, this one consumes a git diff and returns review comments —
the inverse direction of, say, [`aider`](../aider/) or
[`claude-code`](../claude-code/), which start from a change request
and emit a patch. Here you start from a finished patch and ask the
model "what would a senior engineer flag in this?".

The intended invocation is on a feature branch right before you push,
or in CI on a pull request:

```
npx code-review-gpt review              # review staged + unstaged diff
npx code-review-gpt review --ci=github  # post comments on the PR
npx code-review-gpt review --remote main # diff against main
```

Output is grouped per-file with severity tags (`🔴 critical`,
`🟡 warning`, `🟢 nit`), each comment carrying a one-line rationale
and an optional suggested replacement. Reviews are scoped to the
hunks touched by the diff, so context cost stays bounded — this is
the design difference from "feed the whole repo to a model" tools
like [`repomix`](../repomix/) plus [`llm`](../llm/).

## 1. Install footprint

- Node 18+, distributed on npm. Run via `npx code-review-gpt` (zero
  install) or `npm i -g code-review-gpt`.
- Single binary entry; no native deps. ~30 MB once npm resolves the
  dep tree.
- Optional `.code-review-gpt.json` for per-repo overrides (model,
  ignored globs, custom review rubric).
- Bring-your-own API key via `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`.

## 2. License

MIT.

## 3. Models supported

OpenAI (GPT-4o / GPT-4.1 / o-series) and Anthropic Claude (Sonnet,
Opus) out of the box, plus any OpenAI-compatible endpoint by setting
`OPENAI_API_BASE` (Ollama, vLLM, LiteLLM, Groq).

## 4. MCP support

None. The tool is a one-shot reviewer, not an agent loop.

## 5. Sub-agent model

None. One LLM call per file (or per chunked hunk if the file blows
the context window).

## 6. Telemetry stance

Off by default. The only egress is the chosen LLM provider.

## 7. Prompt-cache strategy

None — every invocation is fresh, since the diff itself changes
between calls. Caching would defeat the point.

## 8. Hot keybinds

Not a TUI. CI-shaped output: text to stdout, exit code 0 if no
critical findings, non-zero otherwise (overridable via
`--fail-on=warning|critical|never`).

## 9. Killer feature, weakness, when to choose

**Killer feature.** Diff-scoped review with a CI mode that posts
inline comments to the GitHub PR — install once, get LLM review on
every PR without changing developer workflow. Custom rubrics let
you encode team-specific lint policy ("we forbid `any` in TypeScript
unless commented") that a regex linter cannot express.

**Weakness.** It only sees the diff plus a small slice of
surrounding context. Cross-file invariants — "you changed the
shape of `User` but its callers in `billing/` still expect the old
field" — slip through. Pair with [`symbex`](../symbex/) or
[`repomix`](../repomix/) if you need cross-file reasoning.

**When to choose.** You want PR-time AI review without giving the
model write access. You already use `aider` / `claude-code` to
*write* code; this complements them by *grading* the result.

**Niche vs neighbors.** [`sweep`](../sweep/) is the inverse: it
reads issues and writes PRs. [`promptfoo`](../promptfoo/) grades
prompts, not code. [`aider`](../aider/) and [`mentat`](../mentat/)
edit code, they don't review external diffs. `code-review-gpt` is
the only entry whose input is "a diff someone else wrote" and whose
output is "review comments, no edits".
