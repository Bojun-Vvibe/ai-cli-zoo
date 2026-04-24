# lumen

> Snapshot date: 2026-04. Upstream: <https://github.com/jnsahaj/lumen>
> Last verified version: **v2.22.0** (2026-04-07). License file:
> [`LICENSE`](https://github.com/jnsahaj/lumen/blob/main/LICENSE) — MIT.

A single Rust binary that turns `git` into an AI-narrated workflow:
visual diff viewer with syntax highlighting, AI-generated commit
messages for staged changes, AI explanations for arbitrary commits or
ranges, and an interactive fuzzy-search browser for past commits.

It sits in the same neighborhood as [`opencommit`](../opencommit/) and
[`aicommits`](../aicommits/) but is **read-first** rather than
write-first: the headline command is `lumen explain`, not `lumen
commit`. The catalog already covers the "stage diff → suggest one
message" loop; lumen's slot is "explore, then explain, then maybe
draft."

## 1. Install footprint

- `brew install jnsahaj/lumen/lumen` (macOS / Linux Homebrew tap).
- `cargo install lumen` (any platform with a Rust toolchain).
- Pre-built binaries on the [releases](https://github.com/jnsahaj/lumen/releases)
  page for macOS / Linux / Windows.
- Single static binary, ~5 MB. Optional runtime deps:
  - [`fzf`](https://github.com/junegunn/fzf) — required for `lumen
    explain --list` interactive picker.
  - [`mdcat`](https://github.com/swsnr/mdcat) — required if you want
    Markdown output rendered in colour.
- Configuration via `lumen configure` (interactive) or
  `~/.config/lumen/lumen.config.json`. Per-provider env vars
  (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GROQ_API_KEY`, etc.) are
  read at runtime; nothing is stored in the binary.

## 2. Repo, version, license

- Repo: <https://github.com/jnsahaj/lumen>
- Version checked: **v2.22.0** (2026-04-07).
- License: MIT. License file at the repo root:
  [`LICENSE`](https://github.com/jnsahaj/lumen/blob/main/LICENSE).
- Crate: `lumen` on <https://crates.io/crates/lumen>.

## 3. What it actually does

Five subcommands, each scoped tightly:

- `lumen` (no args) — opens a TUI diff viewer over the current
  working tree with syntax highlighting and inline comment support.
- `lumen draft` — reads `git diff --staged` and emits a
  Conventional-Commits-shaped message. Pipe to `git commit -F -` or
  use `lumen draft | git commit -eF -` to land in `$EDITOR`.
- `lumen explain <ref>` — sends a single commit (or a range like
  `HEAD~5..HEAD`) plus its diff to the LLM and prints a prose
  explanation. `--list` opens an `fzf` picker over `git log` so you
  can pick the commit interactively.
- `lumen list` — fuzzy-search history with the diff inline.
- `lumen operate "<intent>"` — translates a natural-language intent
  into a suggested `git` command (e.g. "undo the last two commits but
  keep the changes staged" → `git reset --soft HEAD~2`). Read-only:
  prints the command, does not run it.

There is no agent loop. Every subcommand is a one-shot LLM call
against a deterministic git input.

## 4. MCP support

None. Tool surface is hand-rolled per subcommand; lumen does not
consume MCP servers and does not expose one.

## 5. Sub-agent model

None. Single LLM call per subcommand.

## 6. Telemetry stance

Off. No analytics in the binary itself. The chosen provider
(OpenAI / Anthropic / Groq / Ollama / OpenRouter / DeepSeek / a custom
OpenAI-compatible endpoint) sees the **diff text** of whatever you
asked it to explain or commit — which by construction includes any
secrets, hostnames, or proprietary code in the diff. For sensitive
trees, point `provider` at a local Ollama (`ollama` provider) or
OpenAI-compatible endpoint and the prompt never leaves the host.

## 7. Token / context strategy

Bounded by what `git` returns:

- `lumen draft` sends `git diff --staged` (one diff, usually small).
- `lumen explain <ref>` sends one commit's diff (also usually small).
- `lumen explain HEAD~N..HEAD` sends a range — the only path where a
  user can blow the context window. Lumen does not chunk or
  summarize; the model either fits the diff or refuses.

Mitigation is the same as every other diff-driven tool in the
catalog: stage less, or pre-filter the diff with `git diff --
':(exclude)package-lock.json'` and pipe to a custom flow. There is
no `--max-tokens` knob.

## 8. Hot keybinds

- TUI diff viewer: `j`/`k` scroll, `n`/`p` next/previous file,
  `c` add comment, `q` quit. Mirrors a `less`/`tig` muscle memory.
- `--list` picker: standard `fzf` keybinds (`Ctrl-J`/`Ctrl-K`,
  `Tab`, `Enter`).

## 9. Killer feature, weakness, when to choose

**Killer feature.** It is the only entry in the catalog that ships
**both** a real TUI diff viewer **and** AI commit/explain in one
binary. Existing tools split the job: `tig` / `delta` are
beautiful diff viewers with no AI; `opencommit` / `aicommits` draft
commit messages but have no review surface. Lumen is the
"`tig` + `aicommits`" merge — review the diff visually, then ask the
model "what is this change actually doing?" without leaving the
terminal.

**Weakness.** Optional dependencies that are *not* optional in
practice: without `mdcat` the explain output is raw Markdown noise,
and without `fzf` the `--list` flow is unusable. Configuration is
JSON, not TOML, so editing by hand is awkward. Provider list is
narrower than [`aichat`](../aichat/) or [`mods`](../mods/) — there
is no Bedrock / Vertex / Cohere / Mistral support out of the box.
No streaming for `explain` output: you wait, then the whole reply
arrives at once.

**When to choose.**
- You already do code review in the terminal and want an LLM
  available without switching to a browser PR view.
- You want one binary that covers `lumen` (review) + `lumen draft`
  (commit) + `lumen explain` (post-hoc archaeology) instead of three
  separate tools.
- You want a *local-only* option (Ollama provider) for a workflow
  that touches every staged diff.

**When to skip.**
- You only want to draft commit messages and never review →
  [`opencommit`](../opencommit/) (git-hook integration) or
  [`aicommits`](../aicommits/) (multi-suggestion picker) is a smaller,
  simpler tool.
- You want the LLM to actually *make* the commit (edit + stage +
  commit in one loop) → [`aider`](../aider/) or
  [`claude-code`](../claude-code/).
- You want a TUI git viewer with no AI dependency at all → `tig` or
  `lazygit` are the canonical answers and lumen does not displace
  them.
- You already pipe `git diff` into [`mods`](../mods/) /
  [`aichat`](../aichat/) and like the Unix-pipe shape — lumen's
  bundled UI is a step *down* in composability.

## 10. Compared to neighbors in the catalog

| Tool | Primary surface | AI scope | Diff viewer | Local-only option |
|------|-----------------|----------|-------------|-------------------|
| lumen | `lumen` (TUI) + 4 subcommands | Review, draft, explain, operate | Yes (built-in TUI) | Ollama / OpenAI-compatible |
| [opencommit](../opencommit/) | `prepare-commit-msg` git hook | Draft commit message only | No | Ollama / OpenAI-compatible |
| [aicommits](../aicommits/) | `aicommits` (one-shot CLI) | Draft commit message only (N candidates) | No | OpenAI-compatible |
| [aider](../aider/) | REPL | Edit + stage + commit | No (line-based diff) | Ollama / 100+ via litellm |
| [mods](../mods/) | Unix pipe | Anything (you compose) | No | OpenAI-compatible |

Decision shortcut:

- "Review the diff and ask the model about it" → `lumen`.
- "Just draft a commit message from staged changes" →
  `opencommit` / `aicommits`.
- "Make the model write the patch and the commit" → `aider` /
  `claude-code`.
- "Ship a one-line shell pipeline that summarizes a diff" →
  `git diff | mods "summarize"`.
