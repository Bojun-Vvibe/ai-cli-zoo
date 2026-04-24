# gptcommit

> Snapshot date: 2026-04. Upstream: <https://github.com/zurawiki/gptcommit>

A Rust `prepare-commit-msg` git hook for authoring commit
messages with an LLM. Same niche as
[`opencommit`](../opencommit/) and [`aicommits`](../aicommits/),
but distinguished by being a **single static Rust binary** with
**per-file diff summarisation** as its loop shape rather than
"send the whole diff in one prompt".

## 1. Install footprint

- `cargo install --locked gptcommit`, or `brew install
  zurawiki/brews/gptcommit`. ~6 MB single binary.
- No Node, no Python, no daemon.
- User config at `~/.config/gptcommit/config.toml`; per-repo
  override at `<repo>/.git/gptcommit.toml`; env-var override via
  `GPTCOMMIT__*` keys.
- `gptcommit install` wires the binary up as a `prepare-commit-msg`
  hook in the current repo (this is the canonical entrypoint).

## 2. Repo + version + license

- Repo: <https://github.com/zurawiki/gptcommit>
- Latest release: **v0.5.17** (tag `v0.5.17`, published 2024-10-12).
- License: **MIT** —
  <https://github.com/zurawiki/gptcommit/blob/main/LICENSE>
- Default branch: `main`.

## 3. Models supported

OpenAI by default; `openai.api_base` accepts any
OpenAI-compatible endpoint (LiteLLM proxy, Ollama, vLLM, LM
Studio, OpenRouter, DeepSeek, Groq). Anthropic Claude is wired
in as a separate `prompt.provider` value. The model is selected
via `openai.model`. No native Gemini SDK — route Gemini through a
LiteLLM proxy if you need it.

## 4. MCP support

None. There is no agent loop — the hook makes one (or several,
see §5) HTTP calls and writes one commit message file. Nothing
for an MCP server to plug into.

## 5. Sub-agent model

Not sub-agents in the multi-role sense, but **per-file
summarisation** is a loop: gptcommit splits the staged diff
file-by-file, prompts the model to summarise each chunk, then
prompts a final "translate these summaries into a Conventional
Commit message" call. This is why it scales to large diffs that
make `aicommits` / `opencommit` truncate or hallucinate — each
prompt stays small. Configurable via `prompt.file_diff` and
`prompt.commit_summary` templates.

## 6. Telemetry stance

Off, no analytics. The binary calls the configured OpenAI /
Anthropic / OpenAI-compatible endpoint and nothing else. The
data-exfil concern is the same as every CLI in this niche: your
staged diff goes to whatever endpoint you configured. Use a local
OpenAI-compatible server for sensitive repos.

## 7. Prompt-cache strategy

None client-side. The per-file summarisation loop does benefit
from provider prefix caching when the same file template is
reused across many files in the same commit, but gptcommit does
not explicitly hint or pin caches.

## 8. Hot keybinds / commands

- `gptcommit install` — installs the `prepare-commit-msg` hook in
  the current repo and prompts for an OpenAI key.
- `gptcommit uninstall` — removes the hook.
- `git commit` — the entire normal git workflow; the hook fires
  automatically and pre-fills `$EDITOR` with the generated
  message.
- `gptcommit config set <key> <value>` / `--local` — read/write
  config; e.g. `gptcommit config set openai.model gpt-4o`.
- `gptcommit config keys` — list all available config keys (the
  documentation surface).
- `gptcommit prepare-commit-msg <commit-msg-file>` — call the
  hook logic directly, useful for CI experiments.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Per-file summarisation as the diff
strategy.** A 4 000-line diff across 30 files becomes 30 small
prompts plus one rollup, instead of one giant prompt that
truncates or loses fidelity. This is the architectural answer to
"my real commits are too big for `aicommits`/`opencommit` to
handle gracefully" — the small-prompt loop generalises to large
PR-sized commits, and the per-file template is configurable. As a
single Rust binary, it also has the cleanest install story in
the niche (no Node, no Python, no Homebrew tap dance).

**Weakness.** **Repo activity slowed sharply since 2024.** Latest
release is `v0.5.17` (2024-10), commit cadence has dropped, and
some configuration knobs lag the OpenAI / Anthropic SDK
roadmap. The per-file summarisation loop costs more tokens than
single-prompt peers — a 30-file commit is 31 calls, not 1. No
multi-candidate picker (`aicommits -g 3` is a different UX).

**When to choose.**

- You want a **single Rust binary** in the commit-message niche
  and refuse to install Node or Python for a git hook.
- You routinely make **large multi-file commits** and your
  current hook truncates or hallucinates — the per-file loop is
  the architectural fix.
- You want **per-repo config** that sits in `.git/gptcommit.toml`
  and is not checked into the working tree.
- You already use **Anthropic Claude** as your default and want
  it as a first-class provider, not an OpenAI-compatible shim.

**When not to choose.**

- You want **multi-candidate picker UX** (→ [`aicommits`](../aicommits/)).
- You want the **richest provider matrix and a polished hook
  installer with `commitlint` config emission**
  (→ [`opencommit`](../opencommit/)).
- You also want **diff viewing and commit explanation** in the
  same binary (→ [`lumen`](../lumen/)).
- You want a **stable, frequently-updated** tool — gptcommit is
  in maintenance mode as of 2026-04.

## Comparable tools in this catalog

- [`opencommit`](../opencommit/) — Node-based peer; richer
  provider matrix and `commitlint` integration; single-prompt
  diff strategy.
- [`aicommits`](../aicommits/) — Node-based peer; OpenAI-only by
  default; multi-candidate picker is the ergonomic.
- [`lumen`](../lumen/) — Rust peer that combines a `tig`-shaped
  diff TUI with AI commit-draft and AI commit-explain in one
  binary.
- [`fabric`](../fabric/) — `write_commit_message` pattern; reuses
  fabric's pattern library instead of adding a dedicated binary.
- [`mods`](../mods/) — `git diff --staged | mods 'write a
  conventional commit message'` is the no-extra-tool answer; you
  lose the hook integration, you gain a one-liner.
