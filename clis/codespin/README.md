# codespin

> Snapshot date: 2026-04. Upstream: <https://github.com/codespin-ai/codespin>
> Pinned commit: `93ba5c2f32348d2e760d5288bae35e3eef8d483b` (last push 2025-01).
> Package version: `codespin` 0.0.276 on npm.
> License: MIT — `LICENSE` at repo root.

A **prompt-file-driven code generator** in Node/TypeScript: instead of a
chat REPL or an autonomous agent loop, you author small `.prompt.md`
files next to the code they target, then run `codespin gen` to expand
each prompt into (or back into) a real source file. The mental model is
"Markdown → code", with the prompt file kept around as the source of
truth so you can re-generate the file when the model, the spec, or the
neighboring code changes.

The right pick when you want a **deterministic, file-first** workflow
that fits into a normal git history without a long-running agent process.

## 1. Install footprint

- `npm install -g codespin`. Node 20+. Single npm package, no native
  binaries, no daemon.
- Per-project config in `.codespin/codespin.json` after `codespin init`.
  Templates live in `.codespin/templates/<name>/` and are user-editable
  Handlebars files — the LLM call uses whichever template the
  `.prompt.md` references.
- Prompt files convention: `path/to/foo.ts.prompt.md` generates
  `path/to/foo.ts`. The two files travel together in git; the prompt
  is intentionally not `.gitignore`d.
- No global state directory required. API keys come from environment
  (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`).

## 2. License

MIT. `LICENSE` is at the repo root, SPDX-detectable. Permissive enough
to use the generated code under any downstream license without taint.

## 3. Models supported

Direct backends:

- **OpenAI** (chat + completion, including GPT-4 / GPT-4o / o-series).
- **Anthropic** Claude family.
- **Google** Gemini via the official client.
- **Any OpenAI-compatible** endpoint (vLLM, llama.cpp server, Ollama,
  LiteLLM router, OpenRouter) by setting `--api openai-compatible
  --base-url <url>`.

Model selection is per-invocation: `codespin gen foo.ts.prompt.md
--model gpt-4o` or `--model anthropic/claude-3.5-sonnet`. There is no
mid-session switching because there is no session — each `codespin gen`
is a one-shot LLM call.

## 4. MCP support

None. `codespin` is a one-shot generator, not an agent with a tool
loop. No `codespin --mcp` server, no client. The closest analog is the
**include directive** in a `.prompt.md` (`{{include path/to/other.ts}}`)
which lets you compose the prompt from other files in the repo — but
that resolves at template-render time, not via tool calls.

## 5. Sub-agent model

None. One prompt file → one LLM call → one output file. You can drive
many in parallel via shell (`find . -name '*.prompt.md' | xargs -P 4
-I {} codespin gen {}`) but the tool itself does not orchestrate.

## 6. Telemetry stance

**Off, no opt-in.** No analytics SDK in the npm package. The only
network egress is to whichever model backend you configured.

## 7. Prompt-cache strategy

Provider pass-through. Anthropic prompt caching kicks in automatically
if the system-prompt slice of a `.prompt.md` is large enough (codespin
sets the standard `cache_control: ephemeral` on the system message).
OpenAI implicit prefix caching applies. There is no codespin-level
cache layer between runs — re-running `codespin gen` on the same
prompt re-issues the LLM call (which is the point: the prompt file is
the artifact, the generation is reproducible-ish).

## 8. Hot keybinds

`codespin` is non-interactive — there is no TUI and no REPL. Useful
sub-commands:

- `codespin init` — scaffold `.codespin/` config + default templates.
- `codespin gen <file.prompt.md>` — generate the target file.
  Flags: `--model`, `--api`, `--base-url`, `--max-tokens`,
  `--template <name>`, `--debug` (prints the rendered prompt before
  sending), `--write` vs `--out -` (default writes to disk; `--out -`
  streams to stdout for piping).
- `codespin go '<inline prompt>'` — one-shot inline prompt without
  authoring a `.prompt.md` file; convenient for "edit this snippet"
  workflows piped through a shell.
- `codespin diff <file.prompt.md>` — re-generate to a temp file and
  show the diff against the on-disk version, without overwriting.
- `codespin commit <file.prompt.md>` — generate, write, and create a
  git commit naming the prompt as the source.
- `codespin batch <glob>` — apply `gen` to every matching prompt file
  with a configurable concurrency.

Prompt-file directives (in `.prompt.md`):

- `{{include path}}` — inline another file's contents into the prompt.
- `{{includeMany "src/**/*.ts"}}` — glob include.
- `{{spec path}}` — include a spec file as the "what to build" block.
- Front-matter: `model:`, `api:`, `temperature:`, `maxTokens:` override
  the per-invocation flags.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **The prompt file is committed alongside the
generated file.** Every generated source file has a `.prompt.md`
sibling that travels with it in git, is reviewable in a normal PR
diff, and is the single source of truth for re-generation. When the
model improves or the surrounding code shifts, you re-run `codespin
gen` and get a new candidate diff — no chat history to replay, no
session to reconstruct, no "the agent already did this and forgot
why". The Handlebars template + include-directive system means a
reasonable codebase ends up with a small library of project-specific
prompt patterns (controllers, route handlers, test scaffolds) that
are themselves versioned files, not chat turns.

**Weakness.** **Stalled maintenance** — last commit on `main` is
2025-01, npm version `0.0.276` from the same window. The 0.0.x
versioning never crossed an API-stability boundary, so the CLI flag
surface is somewhat in flux between minor bumps. No MCP, no agent
loop, no test-and-iterate — if the generated code does not compile,
you re-prompt by hand. Multi-file changes are not first-class: each
`.prompt.md` generates exactly one file, so a refactor that touches
five files is five prompts authored separately.

**When to choose.**

- You want a **file-first, git-native** workflow where the prompt is
  reviewable in PRs and the generation is deterministic-from-inputs.
- You generate **lots of similar files** (CRUD endpoints, test
  scaffolds, DTO classes) and want a templated prompt library that
  is itself versioned code.
- You want **zero ambient agent process** — `codespin gen` runs and
  exits; nothing keeps state between invocations.
- You target **multiple models per project** (cheap model for
  scaffolds, expensive model for harder files) chosen via per-file
  front-matter, not chat history.

**When not to choose.** You want autonomous multi-file edits (`aider`,
`devon`, `plandex`), you want a chat REPL (`aichat`, `mods`), you need
MCP, you need a TUI with diff approval (`devon`, `cline`), or you need
the project on active 2026 release cadence. Pick `aider` (most similar
spiritual cousin with a real agent loop) or `aichat` (chat-first) instead.
