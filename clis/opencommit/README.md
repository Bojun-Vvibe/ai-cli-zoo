# opencommit

> Snapshot date: 2026-04. Upstream: <https://github.com/di-sukharev/opencommit>

A tiny single-purpose CLI that does **one thing**: read your staged
git diff, ask an LLM to write a commit message in a configured
convention (Conventional Commits by default), and either drop it
into your `$EDITOR` or commit directly. It is the most-installed
tool in the catalog's "git workflow glue" niche, and it deserves a
slot precisely because it refuses scope creep — it is not an agent,
not a code reviewer, not a chat client. It is `git commit` with the
message field auto-filled.

The right way to think of it: a `prepare-commit-msg` hook with an
LLM behind it.

## 1. Install footprint

- `npm install -g opencommit` (binary name: `oco`).
- Node 16+ runtime. ~30 MB on disk after npm install.
- Config at `~/.opencommit` (flat key=value file). Per-repo override
  by re-running `oco config set …` while inside the repo (writes a
  local `.env`-style override — verify before relying if you need
  per-repo isolation in CI).
- Optional git hook install: `oco hook set` registers a
  `prepare-commit-msg` hook in the current repo so `git commit`
  itself triggers generation; `oco hook unset` removes it.
- No daemon. Each invocation is a single LLM call.

## 2. License

MIT.

## 3. Models supported

Provider list as of 3.x:

- OpenAI (default, `OCO_AI_PROVIDER=openai`)
- Anthropic (`anthropic`)
- Azure OpenAI (`azure`)
- Gemini (`gemini`)
- Groq (`groq`)
- Mistral (`mistral`)
- Ollama / any OpenAI-compatible local endpoint (`ollama`,
  set `OCO_API_URL` to your local server)
- Flowise / MLX-compatible endpoints via the OpenAI-compatible
  shim — verify before relying.

Model is a free-form string passed through to the provider:
`oco config set OCO_MODEL=gpt-4o-mini` or `claude-3-5-haiku-latest`.
Default is a small/cheap model on purpose; commit messages do not
need a frontier model.

## 4. MCP support

None. Out of scope — opencommit is a one-shot text generator, not
a tool-using agent. There is nothing for an MCP server to plug into.

## 5. Sub-agent model

None. Single LLM call per `oco` invocation. No loop, no planner,
no tool calls. The "agent" is `git diff --staged | LLM | git commit
-m`.

## 6. Telemetry stance

**Off, no analytics SDK.** The binary makes outbound calls only to
the configured model provider. There is an optional anonymous
"thank you" ping on first run that you can disable with
`OCO_GITPUSH=false` / by editing config — verify the exact flag
name on your installed version before relying.

The diff itself is sent to the model provider, which is the obvious
data-exfil concern. If your repo is sensitive, point `OCO_API_URL`
at a local Ollama and use a local model.

## 7. Prompt-cache strategy

None at the CLI layer. Every invocation builds a fresh prompt from
the current `git diff --staged` and sends it. Provider-side prefix
caching (Anthropic, Gemini) does not help meaningfully here because
the diff is the bulk of the prompt and changes every commit.

## 8. Hot keybinds / commands

This is not a TUI; it is a command. Surface area:

- `oco` — generate a message for the current staged diff and prompt
  `[Y]es / [N]o / [E]dit / [R]egenerate` before committing.
- `oco --yes` / `oco -y` — skip the confirm, commit immediately.
- `oco --fgm` — "force-generate-message": print to stdout, do not
  commit. Useful for piping into `$EDITOR` or other tools.
- `oco config set <KEY>=<VAL>` / `oco config get <KEY>` — read/write
  the flat config.
- `oco hook set` / `oco hook unset` — install/remove the
  `prepare-commit-msg` git hook in the current repo.
- `oco commitlint` — generate a `commitlint`-compatible config
  derived from the current convention (so a CI lint job will accept
  what oco produces).

Hook mode is the killer ergonomic: with the hook installed, you
just `git add -p && git commit`, your editor opens with the
generated message already filled in, you tweak and save.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **`prepare-commit-msg` hook integration.**
Once `oco hook set` is run in a repo, you do not change your
workflow at all — `git commit` (no `-m`) opens your editor with a
Conventional-Commits-shaped message already drafted from the staged
diff. You edit if needed, save, done. No new command to remember,
no shell alias to maintain, no copy-paste from a chat window. This
is the single best argument for `opencommit` over "just ask
Claude/GPT to write a commit message" — it removes the workflow
friction entirely.

Secondary feature: the **`--fgm` stdout mode** makes it composable.
Pipe it into `gh pr create --body`, into `glab mr create`, into
`$EDITOR`, into a script that builds release notes from a range of
commits.

**Weakness.** Quality is **diff-only**: opencommit sees the staged
hunks and nothing else. It does not see the issue tracker, the PR
description, the surrounding files, the test output, or the
commit-history convention of the repo. So it generates plausible,
syntactically-correct Conventional Commits — but it cannot tell
you *why* the change was made if the why is not visible in the
diff. For mechanical changes (`refactor: rename foo to bar`,
`fix: handle nil pointer in parser`) it is excellent. For semantic
changes (`feat: add billing dunning flow`) you will rewrite the
body every time.

The other weakness is the obvious one: it sends every staged diff
to a third-party LLM provider unless you point it at a local model.
For repos with secrets-in-diffs risk, the local-Ollama config is
mandatory, not optional.

**When to choose.**

- You commit dozens of times a day and **commit-message hygiene is
  slipping** because writing them is friction.
- Your team enforces **Conventional Commits** (or any
  template-shaped convention) and you want a generator that respects
  the format reliably.
- You want a **single-purpose tool** that you set up once and forget,
  not another chat surface.
- You want **release notes / changelogs** generated from a clean
  history — install opencommit, get clean history, then
  `git-cliff` / `release-please` works much better.

**When not to choose.**

- You want commit messages that **explain intent** beyond what the
  diff shows. Use `aider` (which sees the whole repo and the
  conversation) or just write them by hand.
- You need an **agent** that can also stage the right hunks, run
  tests, or open a PR. That is `aider`, `claude-code`, or
  `opencode` territory — opencommit deliberately does none of that.
- Your repo contains diffs you cannot send to a hosted LLM and you
  do not want to run a local model. Skip.
- You want **PR descriptions, not commit messages.** Different tool
  shape — try `gh copilot pr` or pipe a diff into `aichat`/`mods`/
  `llm` with a custom prompt.

## Comparable tools in this catalog

- [`aicommits`](../aicommits/) — closest sibling; same idea,
  smaller provider matrix, fewer config knobs, but very polished
  defaults.
- [`mods`](../mods/) — if you want `git diff --staged | mods 'write a
  conventional commit'` instead of installing a dedicated tool.
- [`fabric`](../fabric/) — has a `write_commit_message` pattern in
  its library; reuse-the-pattern-library answer to the same problem.
