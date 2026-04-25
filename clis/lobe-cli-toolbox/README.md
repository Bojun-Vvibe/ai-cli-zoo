# lobe-cli-toolbox

> Snapshot date: 2026-04. Upstream: <https://github.com/lobehub/lobe-cli-toolbox>
> Pinned version: **v1.10.1** (monorepo tag; packages version independently — `@lobehub/commit-cli` and `@lobehub/i18n-cli` ship from the same release train).
> License file: [`LICENSE`](https://github.com/lobehub/lobe-cli-toolbox/blob/master/LICENSE).

A two-tool monorepo of small, sharp AI CLIs aimed at the parts of git
hygiene and i18n maintenance that are too repetitive to do by hand and too
context-sensitive to script: **`lobe-commit`** (gitmoji-style AI commit
messages with interactive rewording) and **`lobe-i18n`** (LLM-driven JSON
locale-file translation with diff-only updates).

## 1. Install footprint

- Node 18+; install per-tool: `npm i -g @lobehub/commit-cli` and / or
  `npm i -g @lobehub/i18n-cli`.
- Each binary is small (<5 MB on disk after npm install incl. deps).
- Config lives in `.lobe-i18n.json` / `.lobe-commit.json` at the project
  root, plus per-user defaults in `~/.config/lobehub/`.
- macOS, Linux, Windows native (pure Node; no native bindings).

## 2. License

**MIT**. Single license file at the monorepo root,
[`LICENSE`](https://github.com/lobehub/lobe-cli-toolbox/blob/master/LICENSE),
covering both published packages. Permissive, vendoring-safe.

## 3. Models supported

- OpenAI GPT-4 / GPT-4o / GPT-4.1 (default).
- Any **OpenAI-compatible** endpoint via `OPENAI_PROXY_URL` + `OPENAI_API_KEY`
  in the per-project config — covers Azure OpenAI, Together, Groq, DeepSeek,
  Ollama, vLLM, LM Studio, OpenRouter.
- The model name is per-tool, so `lobe-commit` can use a cheap fast model
  (e.g. DeepSeek Chat for one-line commit messages) while `lobe-i18n` uses a
  stronger one (GPT-4-class for nuance-preserving translation).

## 4. MCP support

No. Both tools are single-shot: read input → one chat completion → write
output. No tool-call protocol surface.

## 5. Sub-agent model

None. Each invocation is one LLM round-trip. `lobe-i18n` does **batch**
internally (groups untranslated keys by target language to amortize the
system prompt), but it's still one model, one loop.

## 6. Telemetry stance

**Off** by default. No analytics endpoint baked into either CLI. Egress is
exactly the OpenAI-compatible endpoint you configured.

## 7. Prompt-cache strategy

`lobe-i18n` keeps the per-language system prompt stable across batches so
provider-side prompt caching (Anthropic / OpenAI cached prefixes) hits on
the system block; only the changing payload (the JSON delta) varies.
`lobe-commit` is too short-lived for cache to matter — each invocation is a
fresh process.

## 8. Hot keybinds (TUI / REPL)

`lobe-commit` has an Inquirer-style interactive flow:
- arrow keys to pick the gitmoji prefix the model proposed,
- `e` (in the message-confirm step) to drop into `$EDITOR` for hand-editing,
- `r` to ask the model to regenerate with a hint,
- enter to accept and `git commit`.

`lobe-i18n` is non-interactive: `lobe-i18n` (no args) reads the config,
diffs source-locale keys against each target, and writes the missing keys
in place. `--help` lists the per-subcommand flags.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Per-task model routing in two-line tools. The same
`OPENAI_PROXY_URL` pattern lets you point `lobe-commit` at a $0.10/M-token
local model and `lobe-i18n` at GPT-4-class for the language work where
nuance matters — without standing up a full agent harness like
[`continue`](../continue/) or [`crush`](../crush/) just to write a commit
message.

**Weakness.** Scope is exactly two tasks. It does not edit code, does not
read your repo beyond the diff (for commits) or the JSON locale files (for
i18n), and will never grow into a generalist agent. If you need
"summarize this PR", reach for [`gptcommit`](../gptcommit/),
[`aicommits`](../aicommits/), or a real agent.

**When to choose.** Your repo is multilingual and the commit log is part
of the documentation. You want both jobs done with one config style and
one billing source, and you don't want a TUI sitting between you and
`git`. Pair with [`aider`](../aider/) or [`opencode`](../opencode/) for
the actual code work; let `lobe-cli-toolbox` own the boring-but-frequent
git + locale chores.
