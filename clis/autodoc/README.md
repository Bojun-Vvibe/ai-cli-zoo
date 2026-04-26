# autodoc

- **Repo:** https://github.com/context-labs/autodoc
- **Version:** unreleased (no tags); pinned to commit `f6069a7` (default branch)
- **License:** MIT (`LICENSE`)
- **Language:** TypeScript
- **Install:** `npm install -g @context-labs/autodoc`

## One-line summary

A toolkit that walks a git repo, generates per-file/per-folder LLM
documentation, and lets you query it from the CLI — "docs as a build artifact".

## What it does

Autodoc is unusual in this catalog: it doesn't try to *write* code, it tries
to *explain* an existing codebase at scale. The flow is three commands:

- `doc init` — configure project name, root path, and which provider/model
  to use (OpenAI by default; any OpenAI-compatible endpoint works).
- `doc index` — recursively walk the repo, summarize each file with the
  LLM, then summarize each folder from its file summaries (a
  hierarchical map-reduce). Output lands in `.autodoc/` as JSON + markdown.
- `doc query "how does auth work?"` — RAG over the indexed summaries with
  citations back to source paths and line ranges.

Cost-aware by design: it estimates tokens before indexing and prompts
before spending, and supports incremental re-indexing on changed files.

```bash
cd my-large-repo
doc init
doc index            # one-time, ~$X estimated, confirms before running
doc query "Where is the rate limiter implemented and what backs it?"
```

## When to choose it

- You're onboarding to a **large unfamiliar codebase** and want a queryable
  map before reading source.
- You want **shippable docs** as a CI artifact (the markdown output is
  versionable) rather than ephemeral chat answers.
- You like the **hierarchical map-reduce** pattern and want a reference
  implementation you can fork.

## When NOT to choose it

- You want a coding agent that *modifies* files — wrong tool; pair autodoc
  with `aider` / `claude-code` / `codex`.
- You want fully local indexing on a tiny budget — the default flow assumes
  a hosted LLM and can be expensive on million-line repos.
- You need active upstream maintenance — the project has been quiet for a
  while; expect to fix small bit-rot when adopting.
