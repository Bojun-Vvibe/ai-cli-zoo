# aicommits

> Snapshot date: 2026-04. Upstream: <https://github.com/Nutlope/aicommits>

The other entry in the "auto-generate a git commit message from the
staged diff" niche. Aicommits is older, smaller, and more
opinionated than `opencommit`: fewer providers, fewer config
levers, but a tighter UX. If `opencommit` is the configurable
power-tool, `aicommits` is the espresso machine that makes one
drink really well.

It earns its own catalog entry because the **interactive
multi-suggestion picker** (`-g 3` returns three candidates,
arrow-key to choose) is a different ergonomic from
`opencommit`'s confirm-or-edit single-suggestion loop, and many
users prefer it.

## 1. Install footprint

- `npm install -g aicommits`.
- Node 14+ runtime. ~15 MB on disk.
- Config at `~/.aicommits` (flat ini-ish `KEY=VALUE` file, one per
  line).
- No daemon. No git hook install command — if you want hook
  behaviour, write the hook by hand pointing at `aicommits` (or
  use opencommit instead).
- Optional companion: `npm install -g @commitlint/cli` if you want
  CI to enforce the convention aicommits is generating to.

## 2. License

MIT.

## 3. Models supported

Out of the box: **OpenAI only.** The CLI hardcodes the OpenAI
client and reads `OPENAI_KEY` from config.

You can point it at any **OpenAI-compatible** endpoint by setting
`OPENAI_API_BASE` in the config (or via env). That covers
Ollama (`ollama serve` exposes an OpenAI-compatible route), LM
Studio, vLLM, llama.cpp's server mode, LiteLLM proxy, Groq,
Together, OpenRouter, DeepSeek, etc. — but every one of those is
"pretend you are OpenAI" mode, not first-class.

If you want native Anthropic / Gemini / multi-provider with
proper SDKs, use `opencommit`. Aicommits is deliberately
single-provider.

## 4. MCP support

None. Same reason as opencommit — it is a single LLM call, there
is no agent loop for tools to plug into.

## 5. Sub-agent model

None. One stdin (the diff), one LLM call, one (or N) suggestion(s)
out. No tools, no loop.

## 6. Telemetry stance

**Off, no analytics.** The binary calls the OpenAI endpoint (or
whatever `OPENAI_API_BASE` points at) and nothing else. No
crash reporter, no anonymous-usage ping.

The data-exfil concern is the same as every CLI in this niche:
your staged diff is sent to whatever endpoint you configured. Use
a local OpenAI-compatible server for sensitive repos.

## 7. Prompt-cache strategy

None. Same as opencommit — diff-bulk-prompts do not benefit from
provider prefix caching, and there is no CLI-side cache layer.

## 8. Hot keybinds / commands

- `aicommits` — generate one suggestion from the staged diff,
  prompt to confirm-or-cancel, then commit.
- `aicommits -g <N>` / `--generate <N>` — generate N suggestions
  (typically 1–5), arrow-key picker to select one, then commit.
  This is the signature interaction.
- `aicommits --type conventional` — switch convention (default is
  a generic short-message style; `conventional` enables
  Conventional Commits).
- `aicommits --exclude <glob>` — exclude paths from the diff sent
  to the model (useful for lockfiles, generated code).
- `aicommits --all` / `-a` — stage all tracked changes first
  (`git add -u`-equivalent), then generate.
- `aicommits config set <KEY>=<VAL>` / `aicommits config get <KEY>`
  — flat config read/write.
- `aicommits hook install` — installs a `prepare-commit-msg` hook;
  some versions ship this, some don't — verify before relying.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **`-g 3` multi-suggestion picker.** Aicommits'
default ergonomic is "show me three plausible commit messages,
let me pick one with the arrow keys, commit". This is genuinely
better than opencommit's single-suggestion confirm-or-edit loop
when you do not yet know what tone you want — you read three,
realise the second one captures intent best, hit Enter, done. It
is the same UX pattern that `gorilla-cli` uses for shell commands
and it works for the same reason: humans are better at *choosing*
than at *editing*.

**Weakness.** **Single-provider by design.** The OpenAI
hardcoding means switching to Anthropic / Gemini / a local model
requires either an OpenAI-compatible shim (LiteLLM proxy, Ollama)
or a fork. If you live in a non-OpenAI provider ecosystem, every
day you use aicommits costs you a config layer that opencommit
does not require.

Also missing relative to opencommit: no built-in
`commitlint` config emitter, no first-class repo-local config
override, smaller convention library (no custom-template config of
the depth opencommit supports). Aicommits is a sharper tool for a
narrower situation.

**When to choose.**

- You **already use OpenAI** for everything else and you do not
  want to wire up another provider just for commit messages.
- You prefer **picking from N candidates** over editing a single
  one — the `-g` flag is the entire reason to choose aicommits
  over opencommit.
- You want **the smallest, simplest tool** in the niche; aicommits
  is ~one screenful of important code, the surface area you can
  audit in an afternoon.
- You are **introducing the tool to a team** that is allergic to
  config — `OPENAI_KEY=sk-... && aicommits` works on day one with
  zero further setup.

**When not to choose.**

- You need **non-OpenAI providers as first-class** (Anthropic,
  Gemini, Bedrock, native Ollama). Use `opencommit`.
- You want a **`prepare-commit-msg` hook** as the primary entry
  point. Opencommit's hook story is more polished.
- You want **deep convention customisation** (custom scopes, custom
  body templates, gitmoji flavors). Opencommit has more knobs.
- You want **stdout-mode for piping** into other tools. Opencommit's
  `--fgm` flag is cleaner; aicommits is built around the
  interactive prompt.

## Comparable tools in this catalog

- [`opencommit`](../opencommit/) — direct competitor; richer
  provider matrix and hook story, less polished
  multi-suggestion UX.
- [`mods`](../mods/) — `git diff --staged | mods 'write 3
  conventional commit messages, numbered'` is the no-extra-tool
  answer; you lose the picker, you gain provider freedom.
- [`fabric`](../fabric/) — `write_commit_message` pattern; reuses
  fabric's existing pattern library instead of adding a dedicated
  binary.
