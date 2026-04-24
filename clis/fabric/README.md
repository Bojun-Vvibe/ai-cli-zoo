# fabric

> Snapshot date: 2026-04. Upstream: <https://github.com/danielmiessler/fabric>
> Binary name: `fabric` (Go binary; the Python `fabric-ai` predecessor is deprecated)

`fabric` is the **prompt-pattern primitive** of the catalog. Every
other entry in this repo treats prompts as private, ad-hoc, baked
into the tool, or buried in a config file. `fabric`'s entire reason
to exist is the inverse: prompts are first-class, public,
version-controlled artifacts called **patterns**, and the CLI's job
is to apply one of them to whatever you pipe in.

```
pbpaste | fabric -p summarize
curl -s https://example.com/blog/post | fabric -p extract_wisdom
yt --transcript https://youtu.be/... | fabric -p create_summary
git diff | fabric -p analyze_pull_request
```

The pattern library (200+ patterns at last count, all
markdown-with-frontmatter, all hand-written) is what people actually
install `fabric` for. The Go binary itself is a fairly thin runner:
load pattern, splice user input, send to provider, stream stdout.
What is special is that you and your team can `git pull` a patterns
directory the same way you `git pull` a dotfiles repo, and every
shell on every machine immediately gets the same prompt library.

This is taxonomically distinct from `sgpt` (which has *roles* — also
file-based prompts, but ~10 of them, scoped to one user, OpenAI-only
function-calling shape) and from `llm` (which has *templates* — also
file-based, but a single-user feature, no community library
convention). `fabric`'s differentiator is the **shared, curated,
versioned pattern library** as the headline product.

## 1. Install footprint

- Single static Go binary, ~30 MB, distributed via GitHub Releases,
  Homebrew (`brew install fabric-ai`), or `go install
  github.com/danielmiessler/fabric/v2@latest`.
- First run: `fabric --setup` — interactive provider config (writes
  `~/.config/fabric/.env`) plus `fabric --updatepatterns` which
  shallow-clones the upstream pattern repo into
  `~/.config/fabric/patterns/`.
- Patterns directory is a plain folder of subdirectories, one per
  pattern, each containing `system.md` (the system prompt) and
  optionally `user.md` (a user-message template). You can edit them
  in place, fork the repo, or point `fabric` at a different patterns
  directory via `--patternsdir`.
- No Python, no Node, no venv. The legacy Python `fabric-ai` package
  on PyPI is a deprecated predecessor — use the Go binary.

## 2. License

MIT.

## 3. Models supported

Multi-provider via per-provider config in `~/.config/fabric/.env`:

- **OpenAI** — GPT-4.1, GPT-5, o-series; baseline default.
- **Anthropic** — Claude Sonnet / Opus / Haiku via the Anthropic SDK.
- **Gemini** — via Google AI Studio key.
- **Groq** — Llama-3.x, Qwen, Mixtral served fast.
- **Ollama** — local models at `http://localhost:11434`.
- **OpenRouter** — single key, hundreds of models.
- **Azure OpenAI** — endpoint + deployment-name routing.
- **Bedrock** — via the AWS SDK.

`fabric --listmodels` prints what is reachable from your current
config. `--model <name>` overrides per-call; `--vendor <name>`
forces a specific provider when the same model name exists in
multiple. The default model is set in the env file and is the
single biggest knob most users tune.

## 4. MCP support

None as a client or server. `fabric` consumes patterns, not tools.
There is no tool-call loop, no MCP server discovery, no JSON-schema
function-calling layer. If you need MCP, this is not your tool —
pick `opencode`, `codex`, `claude-code`, `cline`, or `goose`.

The closest things to "extension" in `fabric` are:

- **Patterns** — the library above. Adding a new pattern is "drop a
  `system.md` in a new subdirectory."
- **Strategies** — meta-prompts that wrap any pattern (e.g.
  Chain-of-Thought, ToT, "act as an expert"). Combinable via
  `--strategy <name>`.
- **Helpers** — small companion CLIs in the same repo: `yt`
  (YouTube transcript fetch), `to_pdf`, `code_helper`. They exist
  to make the Unix-pipe pattern smoother.

## 5. Sub-agent model

None. Each `fabric -p <pattern>` invocation is one provider call.
There is no planner/builder split, no self-reflection loop, no tool
calls. Strategies modify the prompt *before* the call; they do not
introduce a second call.

The composition story is **shell pipes**, not sub-agents. The
canonical example is

```
pbpaste \
  | fabric -p extract_wisdom \
  | fabric -p create_summary \
  | fabric -p improve_writing \
  | tee final.md
```

— three independent provider calls chained by the shell. This is
deliberately the same composition primitive as `mods` and `llm`,
but with a curated shared prompt library instead of ad-hoc prompts.

## 6. Telemetry stance

**Off, with no opt-in.** `fabric` itself sends nothing. The
configured provider obviously sees prompt + completion. Pattern
selection happens entirely on disk. The pattern-update mechanism is
a `git pull` against the public patterns repo and is fully
inspectable.

## 7. Prompt-cache strategy

None at the framework level. Each call is stateless. There is no
on-disk response cache, no prompt-prefix reuse, no session warm-up.

Patterns are short and stable, which means provider-side automatic
prefix caching (OpenAI's, Anthropic's when explicitly enabled,
Gemini's implicit) does kick in for free across many invocations
of the same pattern — a useful but invisible win.

## 8. Hot keybinds

There is no TUI. Relevant ergonomics are CLI flags and shell
integration:

- `fabric -p <pattern> "inline prompt"` — apply pattern with an
  inline user message.
- `cmd | fabric -p <pattern>` — apply pattern with stdin as the
  user message. The dominant idiom.
- `fabric --listpatterns` / `fabric -l` — enumerate installed
  patterns. With 200+ entries, pipe to `fzf` for usability.
- `fabric -y <youtube_url> -p <pattern>` — fetch transcript
  inline, then apply pattern. Equivalent to `yt --transcript ... |
  fabric -p ...` but one fewer pipe.
- `fabric --stream` (default on) / `--stream=false` — toggle
  streaming. Disable when piping into JSON tools that want one blob.
- `fabric -S` / `--setup` — re-run the interactive provider config.
- `fabric -U` / `--updatepatterns` — git-pull the upstream patterns
  directory. Run this monthly; the library moves.
- `fabric --serve` — start a local HTTP server that exposes the
  pattern set as REST endpoints. Useful for IDE integrations or
  in-house webhooks. Out of scope for terminal-only use.

The single most important "keybind" is your shell's `fzf`-binding
on top of `fabric -l`:

```sh
fp() { fabric -p "$(fabric -l | fzf)"; }
```

— now `pbpaste | fp` interactively picks a pattern.

## 9. Killer feature, weakness, when to choose

**Killer feature.** A **shared, version-controlled, hand-curated
pattern library** treated as the product. Once your team agrees on
"we use `extract_wisdom` for talk notes, `analyze_paper` for arxiv
PDFs, `create_summary` for meeting transcripts, `improve_writing`
for first drafts," every member gets identical behavior on every
machine by `git pull`-ing the same patterns directory. This is
prompt engineering as a *library* problem rather than as a
*per-tool-config* problem, and nothing else in the catalog frames it
that way at this scale.

**Weakness.** Three:

1. **Patterns are a library, not a runtime.** `fabric` cannot edit
   files, run shell commands, call MCP tools, loop on errors, or
   maintain state across calls. If your problem needs any of those,
   `fabric` is the wrong layer; you want an agent CLI with tool
   calls. People who try to make `fabric` do agent work end up
   reinventing `opencode`/`codex` poorly.
2. **Pattern quality varies.** 200+ patterns means the long tail is
   uneven — some are tight one-page prompts that earn their keep,
   some are sprawling unmaintained drafts. Treat the library like
   any large open-source plugin ecosystem: skim before relying.
3. **Provider abstraction is shallow.** `fabric` speaks each
   provider's native API directly rather than going through LiteLLM
   or similar. New providers land when someone PRs them. The
   `--vendor` / `--model` UX is fine; the breadth of supported
   providers is narrower than `llm` (plugin model) or `aichat` (20+
   built in).

**When to choose.**

- You want a **team-shared prompt library** that everyone gets via
  `git pull`, with the CLI as the runner.
- You think in **Unix pipes**: `pbpaste | fabric -p X | tee out.md`
  is your natural composition shape.
- You consume a lot of **YouTube talks, arxiv PDFs, blog posts,
  meeting transcripts** and want a small set of repeatable
  reductions over them.
- You want a single **stable Go binary** (no Python venv, no Node
  toolchain) that does prompt-application well and nothing else.
- You want the **patterns themselves to be your product** — the
  library is forkable; many teams maintain a private fork on top of
  upstream.

**When not to choose.**

- You need to **edit project files**. Use `aider`, `opencode`,
  `codex`, `claude-code`, or `cline`.
- You need **MCP**. `fabric` has none.
- You need **agent loops with tool calls**. Pick any agent-class
  entry; this is the wrong taxonomic layer.
- You want a **queryable SQLite log of every call**. Use
  [`llm`](../llm/) — `fabric` does not log by default and adding
  logging is your job.
- You want **shell-command generation with execute prompt**. Use
  [`shell-gpt`](../shell-gpt/) or [`tgpt`](../tgpt/) — `fabric` is
  for prose / structured-output transformations, not "give me the
  rsync flags."
- You want **zero-config no-key**. Use [`tgpt`](../tgpt/) — `fabric`
  requires you to bring a provider key on first run.
