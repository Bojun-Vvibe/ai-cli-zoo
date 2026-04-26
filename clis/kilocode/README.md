# kilocode

- **Repo:** https://github.com/Kilo-Org/kilocode
- **Version:** `v7.2.24` (latest release, 2026-04-25)
- **License:** MIT (`LICENSE`)
- **Language:** TypeScript (VS Code extension + companion CLI)
- **Install:** Search "Kilo Code" in the VS Code marketplace, or
  `code --install-extension kilocode.Kilo-Code`. A standalone
  `kilo` CLI ships from the same repo via `npm i -g @kilo-code/cli`
  for headless / pipeline use.

## One-line summary

A community fork lineage of the Roo Code / Cline VS Code agent that
merges **Architect → Code → Debug → Ask modes**, an extensible
**marketplace of Skills, Modes, and MCP servers**, and a no-config
zero-cost OpenRouter-backed entry path into a single editor-resident
agentic coding extension — with the exact same tool-call surface
exposed to a headless CLI for CI use.

## What it does

Kilo Code is the merged successor to Roo Code (which itself forked
Cline). The pitch is "all the agent modes from the Cline / Roo
lineage, plus a curated marketplace, plus a free trial credit so you
don't have to pick an API key before you can try it":

- **Mode-switched agent**: Architect (planning), Code (edits +
  patches), Debug (read logs / re-run / propose fix), Ask
  (answer-only, no edits) — same model, different system prompts and
  tool restrictions per mode. Mode is a top-bar dropdown, not a slash
  command.
- **Marketplace**: `Kilo-Org/kilo-marketplace` is a curated index
  of **Skills** (named multi-step procedures with their own prompts +
  files), **Modes** (custom system prompts + tool allowlists), and
  **MCP servers** (one-click install + auth). The extension's
  marketplace pane lets you install any of them without leaving the
  editor.
- **Provider-agnostic**: Anthropic, OpenAI, Gemini, Mistral, Bedrock,
  Vertex, Ollama, LM Studio, OpenAI-compatible (LiteLLM, vLLM, any
  proxy), plus a built-in OpenRouter path that ships **$20 of free
  credits on first install** so you can try it before configuring a
  key.
- **Approve-each-step UX**: every file edit, terminal command, and
  MCP tool call is rendered as a card with an Approve / Reject /
  Auto-approve-this-pattern button, inherited from the Cline trust
  model. Auto-approve scopes are per-workspace and persisted.
- **Companion CLI** (`@kilo-code/cli`): runs the same agent loop
  headlessly with the same tool surface, configured via
  `.kilocode/config.yaml`. Useful for CI ("apply this skill to the
  diff, fail the job if it edits anything outside `src/`") and for
  driving the agent over SSH on a remote dev box.

```bash
# editor path
code --install-extension kilocode.Kilo-Code
# headless path
npm i -g @kilo-code/cli
kilo run --mode architect "design a migration from X to Y"
```

The extension stores chat history per workspace in
`.kilocode/history/`, and skills + custom modes in
`.kilocode/skills/` / `.kilocode/modes/` so they version-control
naturally with the project.

## When to choose it

- You liked Cline / Roo Code's approve-each-step trust model and
  want the **merged feature set with active maintenance** (Roo and
  Cline both upstream now flow into Kilo Code as well as their own
  releases).
- You want a **marketplace of pre-built skills and MCP servers**
  rather than authoring every prompt yourself — and you want
  installs to be one click, not "clone repo and edit settings.json".
- You want **the same agent in the editor and in CI** with one
  config file, instead of running Cline interactively and a
  different agent (aider / opencode) in pipelines.

## When NOT to choose it

- You don't use VS Code as your primary editor — the extension is
  the main surface, and the headless CLI is younger and lags the
  feature set.
- You want a single-vendor / single-model integration with no
  marketplace surface — `claude-code` or `gemini-cli` are simpler
  and have less moving config.
- You need a **fully audited supply chain** for skills you install
  — the marketplace is curated but skills can ship arbitrary
  prompts and MCP server configs; treat installs the same way you'd
  treat an npm dependency.
